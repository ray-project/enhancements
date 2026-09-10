# REP: Application-Level Fault Tolerance for Ray Data

Authors: David Dai, Ayush Kumar

Date: Aug 3, 2026

## Summary

### General Motivation

As the scale and "spottiness" of Ray Data workloads increase, node loss becomes routine rather than exceptional. Today, every lost node makes a running Ray Data job measurably worse: spilling spikes, the pipeline stalls, throughput degrades, and the job frequently ends in an `ObjectLostError`.

The root cause is a schism between two independent schedulers. When a node is lost to spot preemption or failure, **Ray Core's lineage reconstruction** resubmits the producer tasks needed to rebuild lost objects. That resubmission happens entirely inside Core, below and outside of **Ray Data's streaming executor**, which owns operator queues, backpressure, and resource accounting for the same workload. Specifically:

1. **Split-brain scheduling.** Lineage reconstruction's rescheduling mechanism lives outside Ray Data's scheduling and resource accounting layer. Reconstruction tasks are not backpressured and are not counted against Data's resource budgets, which leads to unexpected cluster resource usage and significant performance degradation (the excessive spilling users observe).
2. **No actor pool awareness.** Ray Core has no representation of an actor pool that would let a failed actor task run on a *different* actor in the same pool; it requires the task to re-run on the same actor. When there aren't enough resources to bring that actor back, the workload waits for a long time on retry.
3. **Poor debuggability.** Debugging reconstruction issues originating from Data pipelines is difficult, because accounting and lineage information about lost objects in Core cannot easily be reconciled with the corresponding operators or execution state in Data.

This proposal unifies reconstruction with the happy-path scheduling mechanism by **moving lineage tracking into the application layer (Ray Data)** and letting the `StreamingExecutor` schedule reconstruction tasks and fresh tasks through the same path. Reconstruction then inherits Data's backpressure, resource budgets, actor pools, and observability for free.

#### Goals

- **Stability**: Ray Data jobs should complete even under frequent node preemptions and node failures in large batches.
- **Predictability**: Ray Data jobs should have predictable, consistent resource utilization under failures. Failure should not cause excessive spilling.
- **Debuggability**: Reconstruction should be fully observable — alerting on bad FT behavior before customers hit it, plus enough logging, metrics, and runbook guidance for an on-call engineer to debug reconstruction.

#### Success criteria

1. Guaranteed job completion under node loss.
2. Stable resource utilization under node loss.
3. Feature parity with the fault tolerance coverage that Ray Core lineage reconstruction provides today.
4. Application-level visibility into recovery progress under node loss.

#### Secondary goals

- Fault tolerance performance (recovery time), as opposed to correctness/completion.

#### Non-goals and assumptions

- **Exactly-once is not provided.** The mechanism provides *at-least-once* execution, matching Core's behavior today.
- **UDFs are assumed to be deterministic.** A reconstructed task that runs a non-deterministic UDF will produce different output than the original.
- **Order preservation is not a requirement under failure.**
- Driver failure is out of scope for this proposal; it is handled by the complementary Ray Data driver checkpointing work (see [Relationship to checkpointing](#relationship-to-checkpointing-and-stable-store-mechanisms)).

#### Example workloads

- All Ray Data workloads depend on object store accounting to determine whether to backpressure a stage, and 
  CPU, memory, and GPU resource accounting to ensure good utilization across the cluster. Whenever Core lineage
  reconstruction triggers, the tasks are submitted without Ray Data's coordinator's knowledge. We've observed 
  that this leads to Ray Data's resource accounting to stray from the actual cluter utilization, leading to
  periods of poor performance.
- Data processing workloads with preemptible GPU nodes commonly have actor pools executing the gpu stages. 
  When the nodes are preempted, under existing Ray Core lineage reconstruction, task must be reexecuted on the same actor. 
  Thus, we need to wait for a new node to come up when other actors in the pools are available. Data fault tolerance addresses
  This issue by removing the constraint that actor task must be resubmitted to the same node since Ray Data is aware of the 
  actor pool.
- Data processing workloads that are autoscaling can choose to scale up or down an individual stage depending on utilization.
  Again, with actor pool stages, a failed actor task must be resubmitted to the same actor. However, if that stage was scaled
  down, the actor may not have enough resources to be scheduled, leading to potential reconstruction time outs.

### Should this change be within `ray` or outside?

Within `ray`. The bulk of the change lives in `ray.data` (a new lineage tracker owned by the `StreamingExecutor`, plus changes to operator input/output handling and task submission). It also requires a small amount of Ray Core surface: a supported way to disable lineage pinning/reconstruction *for Ray Data's objects only*, without changing behavior for the rest of the cluster.

## Stewardship

### Required Reviewers

TODO: fill in Ray Data and Ray Core committers before marking ready for review.

### Shepherd of the Proposal (should be a senior committer)

TODO

## Design and Architecture

### Terminology

| Term | Definition |
|---|---|
| **Root / seed task** | A task with no upstream dependencies. These tasks can always be resubmitted without waiting on anything else to be resolved. |
| **Fan-out stage** | A stage where a single operator task may produce multiple blocks consumed by multiple consumer tasks. |
| **Fan-in stage** | A stage where a single operator task may consume multiple blocks produced by multiple producer tasks. |
| **Lineage** | The dependency chain for a task — the chain of tasks that led to the current task being submitted. |
| **Lineage completion** | A lineage is *complete* when the final task in the chain produces no output needed by any downstream task. |
| **Data Task ID** | A stable Data-level task identity that is preserved across reconstruction attempts (see [Data Task IDs](#data-task-ids)). |

### Background: Ray Core lineage reconstruction

For the purposes of a Ray Data workload, Core's lineage reconstruction can be summarized as:

1. A task attempts to fetch an object it depends on. If the object is not immediately available, it waits.
2. The owner of the object (the driver, in this case) determines that the node holding the object is gone and begins reconstruction.
3. Core looks up which task produced the object and attempts to resubmit that producer task.
4. If an input of the producer task is itself lost, recurse from step 1 for that producer; otherwise resubmit.

This guarantees the lost object is reconstructed at least once, and in the common case (no duplicated submission) exactly once per failure. It assumes UDF determinism.

Critically, **this all happens inside Core, independent of application code**: UDFs are re-executed with no corresponding task submission at the Ray Data level. That is precisely the property this REP changes.

### Overview of the proposed design

```
             Today                                    Proposed
  +---------------------------+            +---------------------------+
  |   Ray Data                |            |   Ray Data                |
  |   StreamingExecutor       |            |   StreamingExecutor       |
  |   - operator queues       |            |   - operator queues       |
  |   - backpressure          |            |   - backpressure          |
  |   - resource budgets      |            |   - resource budgets      |
  +---------------------------+            |   - LineageTracker  <--+  |
              |  fresh tasks                |   - reconstruction     |  |
              v                             |     plans              |  |
  +---------------------------+            +---------------------------+
  |   Ray Core                |                   |  fresh + reconstruction
  |   - lineage pinning       |                   |  tasks (one path)
  |   - reconstruction  ------+--> resubmits      v
  |     (invisible to Data)   |    tasks    +---------------------------+
  +---------------------------+   *outside* |   Ray Core                |
                                  Data's    |   - lineage pinning OFF   |
                                  scheduler |     for Data objects      |
                                            +---------------------------+
```

The design has four parts:

1. **Lineage tracking** — the `StreamingExecutor` maintains a task dependency graph so it can trace back from a failed task to the seed tasks needed to reproduce it.
2. **Reconstruction planning** — on failure, the executor builds a plan describing exactly which upstream tasks must re-run and which of their outputs are actually needed.
3. **Scheduling** — reconstruction tasks are submitted through the executor's normal submission path, so they are subject to backpressure and resource budgets like any other task.
4. **Garbage collection** — lineage metadata is dropped once outputs are durably consumed, either automatically (sink/materialization) or via an explicit user-facing API.

### API Design

#### Feature flags

| Flag | Default |
|---|---|
| `RAY_DATA_RECONSTRUCTION` | `OFF` |
| `RAY_LINEAGE_PINNING_ENABLED` | `ON` |

We add a new feature flag, `RAY_DATA_RECONSTRUCTION`, that toggles Ray Data reconstruction. It is **off by default** while development completes.

Ray lineage pinning (Core reconstruction) **must be disabled for Ray Data reconstruction to work** — otherwise both mechanisms will attempt to recover the same objects. Lineage pinning is currently controlled only by an environment variable, so enabling the Ray Data fault tolerance flag must also disable lineage pinning.

> **Open ask (Ray Core):** we need a way to disable lineage reconstruction *only for Ray Data workloads*; the rest of the cluster should be unaffected. A job-level configuration is one candidate, but the right configuration surface still needs design. This is the main Core-side dependency of this REP.

#### Public API: `block_consumed`

```python
def block_consumed(block: ObjectRef) -> None: ...
def block_consumed(block_id: str) -> None: ...
```

`block_consumed` tells Ray Data that a block has been committed and its lineage can be garbage collected — the block will never be reconstructed again. When every output block of a task has been marked consumed, that task is *lineage complete*.

`ray.train`'s `dataset.state_dict` should invoke this API under the hood for all blocks that it saves.

#### Internal API: `clear_block_lineage`

```python
def clear_block_lineage(data_task_id: str, output_index: int) -> None: ...
```

Used internally when Ray Data itself can observe that a block has been fully materialized. `block_consumed` resolves the object ref to a `(data_task_id, output_index)` pair and calls `clear_block_lineage` underneath.

### Lineage tracking

To reconstruct a failed task we must know the dependency chain that produced its inputs: for each task, which arguments it needs and which task produced them. We considered several designs. The chosen design is described below; the alternatives are in the [Appendix](#alternative-designs-considered).

#### Plan-based reconstruction

To avoid redundant work on fan-out and to know when mappings can be erased, we track the full lineage as a graph. Each node is a submitted task; each directed edge is a dependency (a task depends on another when it takes one or more of that task's outputs as an argument).

The graph is built during normal execution:

1. When the `StreamingExecutor` submits a task, add a node for that task.
2. Group the submitted task's arguments by producer, and draw an edge from each producer to the submitted task, recording which of the producer's outputs are needed. Edges are implemented bidirectionally so the graph can be walked in both directions.

```
  seed_0 ──► map_0 ──┬──► agg_0
                     └──► agg_1
  seed_1 ──► map_1 ──┘
```

**Plan construction.** When a task fails, the executor picks up the error and constructs a *reconstruction plan* describing exactly the steps needed to rebuild the failed task:

1. Generate a new unique `plan_id` for this reconstruction attempt, associated with the failed task.
2. For the current node, build a mapping from downstream tasks participating in this attempt to the output object indices they depend on. A downstream task participates if it carries this `plan_id`. Note the failed task itself has no downstream dependencies in the same attempt.
3. Record on the current node a new mapping from `plan_id` to the map built in step 2.
4. Walk up to the parents of the current node and repeat steps 2–3 until reaching the seed tasks.

**Resubmission.** Once the plan is built, every seed task reached during plan construction is resubmitted, tagged with the `plan_id`:

1. Each time a resubmitted task produces an output, look up the plan on that node by `plan_id` to check whether the object is needed. If a downstream task with the same `plan_id` needs it, resubmit that downstream task with the object once all of its arguments are resolved. Otherwise ignore the output and let it be GC'd immediately.
2. When a resubmitted task completes, remove the plan entry from its node to signal that this task's part of the reconstruction is done.

This repeats until the originally failed task completes.

**Failure mid-reconstruction.** If a resubmitted task itself fails, we do *not* create a new plan. We re-run the existing plan starting from the failed resubmitted node.

**Garbage collection.** When a lineage completes, walk back up the dependency graph from the terminating node; for each node with no remaining children, remove it from the graph.

**Pros**

- Reconstruction tasks go through the `StreamingExecutor`'s queue and are backpressured like normal tasks — no Core/Data split brain.
- Reconstruction tasks are new Core tasks, so with actor pools they can run on any actor in the pool.
- Supports all Ray Data workloads (fan-in and fan-out) without reconstructing tasks unrelated to the failure.
- Lineage metadata can be garbage collected cleanly on lineage completion.

**Cons**

- A new plan is created per fresh task failure. If two reconstructions are triggered simultaneously and share a common upstream lineage, that shared lineage is reconstructed twice.
- Plan metadata scales with the number of simultaneous reconstruction attempts.
- *Mitigation idea:* batch reconstruction triggers (e.g. defer until the end of a scheduling window, or until resources are otherwise underutilized) so overlapping attempts can be merged.

#### Decision

We recommend **plan-based reconstruction**: it is the only option that covers all Ray Data topologies, avoids reconstructing unrelated work, and supports clean garbage collection, at the cost of duplicated work across simultaneous overlapping reconstructions — which can be mitigated by batching.

### Scheduling reconstruction tasks

#### Data Task IDs

Since the reconstruction tasks submitted from Data's `StreamingExecutor` are technically new Ray tasks, they do not preserve their original Ray task ID. We need a stable identity that allows us to differentiate between tasks that were run for the first time, and tasks that are re-run as part of reconstruction. For this reason, we define a stable **Data Task ID** which is preserved across reconstruction attempts.

Data Task ID is defined as:

```
data_task_id = "{op_id}:{task_index}"
```

where `op_id` is the physical operator's UUID and `task_index` represents the value of the counter which tracks the fresh task index of an operator stage.

#### Triggering reconstruction

As mentioned above, we catch an `ObjectLostError` while processing completed tasks in the `StreamingExecutor`. The scheduler might catch and trigger multiple lineage reconstructions in the same scheduling tick. The triggering task might also itself be a "reconstruction attempt", but is treated the same way as a regular task failure except that we do not create a new task node for it.

#### Resubmitting seed tasks

After the failed task is registered with the lineage tracker, we trace the task node graph, returning the set of seed task IDs the plan needs to re-execute. The current failed task is marked aborted. We then re-submit the seeds and continue, without failing the pipeline.

For each task in the chain of reconstruction tasks, we are careful to only propagate the blocks that are needed to reconstruct a lost object, or are fresh blocks. These dependencies are accounted for in the lineage tracker by tracking Data Task ID and object index for every submitted task. This means that any output blocks that are not needed to reconstruct a lost object, or are not fresh blocks, are pruned away. Also, reconstruction blocks are supposed to be withheld by the producing operator until a child reconstruction task that consumes them can be scheduled, i.e. all the required input blocks for the particular reconstruction task are produced by its parents.

#### Batching reconstruction plans

In a naive implementation, every lost object opens its own plan and re-injects every seed it traces to. A node death that destroys `N` blocks descended from one `ListFiles` seed task therefore re-runs that single task `N` times, pruning a different set of output indices in the chain on each execution.

We resolve this wasted effort by letting later plans attach to a seed resubmission that is still queued. Critically, under our approach, we can only batch seed reconstruction plans for a seed task that is still queued, rather than one that is already in flight. This is because with a seed task in progress, an object that is required by a new task failure that traced to the same seed might have already been pruned, and batching here would lead to missing blocks.

The process of batching reconstructions follows:

1. When a plan re-submits a seed task, the seed operator records each seed Data Task ID, and a set of reconstruction plans that this seed task provides outputs for. The operator is responsible for tracking this data because it knows which seed tasks are queued and which are already scheduled.
2. A new plan first checks the seed operator to see if the seed task it traces to has already been queued, and if so, simply joins the set of pending reconstructions for this seed task.
3. When the seed task is dispatched and starts producing outputs, it needs to check if any of the pending reconstruction plans require each object that is produced. That decides if the object is pruned or held for the downstream reconstruction task.
4. As in the case without batching, we withhold reconstruction blocks, except we have to separately track them for each reconstruction plan and release each plan group separately. This prevents the blocks from different reconstruction plans from interfering with each other in fan-in tasks downstream, to ensure deterministic reconstruction.

#### Resource management

Because reconstruction now runs inside Ray Data, we get resource management and scheduling control implicitly. Reconstructed blocks live in the same operator queues as fresh blocks, which means reconstruction is ratcheted by the existing backpressure mechanism and cannot flood the cluster with work. Existing resource budget checks apply before an operator takes on reconstruction tasks, exactly as they do for fresh tasks.

Keeping unified queues for fresh and reconstruction blocks is an intentional decision, to avoid duplicating operator input/output handling in the codebase. It does introduce complexity: we must ensure reconstruction blocks do not bundle with fresh blocks, and that a reconstruction task receives exactly the reconstruction blocks its plan requires as inputs.

Pros and cons of reusing Ray Data's existing operator queues, resource budget, and scheduling policy for reconstruction:

**Pros**

- No need to re-implement complex mechanisms such as backpressure or operator queuing for reconstruction tasks and blocks.

**Cons**

- Admission control becomes more complicated: operator queues are now heterogeneous (fresh and reconstructed blocks), and reconstruction tasks must be scheduled only with their corresponding inputs.
- We lose some observability into the progress of reconstruction tasks specifically, since they share queues with fresh work.

### Data–client interface and lineage garbage collection

Ray Data pipeline outputs are consumed in several ways:

- Materialize to driver: `take`, `take_batch`, `show`, …
- Aggregations: `sum`, `min`, `max`, `mean`, …
- Metadata queries: `schema`, `columns`, `num_blocks`, …
- Dataset iteration: `dataset.iterator()`, `streaming_split`, …
- Write to sink: `write_datasink`
- Materialization: `materialize`
- *(Experimental)* Materialization into other frameworks: `to_random_access_dataset`

These fall into two categories for garbage collection purposes: **block refs fate-share with the pipeline**, and **block refs outlive the pipeline**.

> **Scope:** we plan to support write-to-sink and `streaming_split` as first-party. Supporting all other interfaces is lower priority.

#### Block refs fate-share with the pipeline

Blocks produced by the last operator are materialized — written to a sink, aggregated, or materialized to the driver. Once materialization completes for a block, it is no longer needed.

When a block is materialized, we invoke `clear_block_lineage(data_task_id, output_index)` so the lineage tracker can drop its state; the block will not be reconstructed. Concretely for writes: each time a write task completes, we invoke `clear_block_lineage` on each of its input blocks.

#### Block refs outlive the pipeline

Blocks produced by the last operator have their refs passed downstream, outside the pipeline. Here Ray Data cannot know when a block is safe to GC unless the user explicitly indicates the block is fully consumed and will never be needed again — that is what the public `block_consumed` API is for.

For blocks to remain reconstructable, **the streaming executor must remain pinned until all output blocks of the last stage are marked consumed**, since the executor holds the lineage. `block_consumed` resolves the block to its Data Task ID and output index and calls `clear_block_lineage`.

### Data–Train interface

Ray Train consumes Ray Data output through the iterator interface, so reconstruction must interoperate with the dispatch loop.

```python
# Data driver process
def dispatch_loop():
    while not done:
        for worker in train_workers:
            block_ref: ObjectRef = executor.get_next_block()
            if needs_data(worker):  # gates based on prefetch backpressure
                worker.receive_block.remote(block_ref)

# Train worker process
block_queue = queue.Queue()

class RayTrainWorker:
    def receive_block(self, block: pyarrow.Table):
        block_queue.put(block)

# Running on another thread
def user_training_loop(self):
    # constructs batches from the next block in `block_queue`
    for batch in ds.iter_batches(...):
        do_training_step(batch)
```

A complication is that **Train's batches do not line up with Data's blocks**. Each trainer must track the index it is at within a block; given that, reconstruction need not be more complex than the block-level design above.

Failure classes Train wants to tolerate, and where this design lands on each:

| Case | Description | Status |
|---|---|---|
| **T1** | Coordinated checkpoint | Relies on Data block-based checkpointing lining up with Train's checkpointing. No reconstruction needed. |
| **T2A** | Single Train worker death with full worker-group restart | Open: if the Train worker dies but all Data workers are alive, the Data outputs may not need reconstruction and could be re-fetched directly. |
| **T2B** | Single Train worker death with in-place healing | Open: relies on Data identifying the exact blocks lost by Train so it can determine what to reconstruct. |
| **T4** | Data worker / node loss | Data reconstruction as proposed here should be sufficient. |

The T2A/T2B interfaces are **open design work** and are called out as follow-on in [Follow-on Work](#follow-on-work).

### Relationship to checkpointing and stable-store mechanisms

We have not benchmarked against a checkpointing or write-ahead-logging approach, but we evaluated the options at a high level.

Data processing systems come in stateful and stateless flavors. Stateful implementations almost always checkpoint, because there is no way to reconstruct state without re-executing the pipeline from the start. Stateless implementations (Spark, Ray Data) have typically chosen reconstruction. The reason is that either way, re-executing part of the lost work is unavoidable; reconstruction is simply an optimization where only the branch that was actually lost is recomputed, rather than rolling back to a checkpoint and redoing everything after it. Writing to a stable store additionally incurs the cost of writing to S3 or similar, which in-cluster reconstruction avoids.

We should explore these options for completeness if time allows, but high-level evaluation favors reconstruction.

**Ray Data driver checkpointing** is the one checkpointing mechanism we will use. It is complementary rather than competing: it targets *driver* failures, whereas the design in this REP targets *worker node* failures.

### Failure Analysis

<table>
  <thead>
    <tr>
      <th colspan="2">Failure</th>
      <th>Behaviour</th>
      <th>Outcome</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th colspan="4" align="left">Worker death when task in flight</th>
    </tr>
    <tr>
      <td>1a.</td>
      <td>Worker dies with task in flight, before task can emit a single output</td>
      <td>Retry of the task is handled by Ray Core. Ray Data submits every task with max_retries=-1 and RaySystemError in retry_exceptions (remote_fn.py), so a worker death mid-task is retried transparently to Data lineage tracking. To lineage tracker, this is a fresh task stored in the TaskNode graph, with no dependencies yet</td>
      <td>Ready blocks processed after task retry as if with fresh tasks. Transparent to Data LR</td>
    </tr>
    <tr>
      <td>1b.</td>
      <td>Worker dies with task in flight, after task has emitted k blocks</td>
      <td>Lineage tracker has registered the k blocks pulled to the driver, along with any downstream tasks that have taken the ready blocks. On worker failure, the task is aborted by Data. The task chain is re-submitted starting with the seed for this task. On output blocks produced for these tasks, only the blocks that are required to reconstruct a lost object are re-emitted to downstream tasks</td>
      <td>All blocks are consumed exactly once. Data LR uses its metadata to check which blocks are OBJECT_REUSED , which are stored before they can be emitted together for downstream tasks to reconstruct lost objects. We drop any blocks with status OBJECT_PRUNED</td>
    </tr>
    <tr>
      <th colspan="4" align="left">Worker death after task completion</th>
    </tr>
    <tr>
      <td>2a.</td>
      <td>Blocks located on producer node, and are already fetched by downstream task</td>
      <td>A copy lives on the consumer's node. Lineage tracker records the task submission of the downstream task and the dependency edge between the failed task and the downstream task. Since the copy exists, the downstream task succeeds as is</td>
      <td>Downstream task executes as normal, blocks not lost, lineage tracker no-op</td>
    </tr>
    <tr>
      <td>2b.</td>
      <td>Block queued as an ObjectRef in a downstream map operator's input queue</td>
      <td>The consumer dequeues block, fails with ObjectLostError before producing anything.</td>
      <td>Lineage tracker traces the failed task, re-submits the chain from the seed. The consumer re-runs under its original id and its outputs are OBJECT_NEW.</td>
    </tr>
    <tr>
      <td>2c.</td>
      <td>Block queued for the driver (iter_batches, take)</td>
      <td>The driver is not a task and registers nothing. ray.get in the consumer raises.</td>
      <td>ObjectLostError thrown to user code.</td>
    </tr>
    <tr>
      <th colspan="4" align="left">Failure during LR in linear pipeline</th>
    </tr>
    <tr>
      <td>3a.</td>
      <td>Worker dies during reconstruction task in flight</td>
      <td>Lineage tracker re-uses the same reconstruction plan ID. Failed task is aborted and the lineage tracker re-submits the seed, pruning objects as in 1b.</td>
      <td>All blocks are consumed exactly once. Existing ready reconstruction blocks are not re-emitted.</td>
    </tr>
    <tr>
      <th colspan="4" align="left">Failure during LR during task fan out</th>
    </tr>
    <tr>
      <td>4a.</td>
      <td>Only one child task fails because its input is lost due to a failure / node death</td>
      <td>Reconstruction triggered, only the index that the child task consumed are OBJECT_REUSED, all other indices are OBJECT_PRUNED</td>
      <td>After re-submission of the seed, only the failed child branch is considered in the reconstruction, the other branches make progress as usual</td>
    </tr>
    <tr>
      <td>4b.</td>
      <td>Several children lose inputs at once, fail together</td>
      <td>Batching reconstructions: The lineage tracker tries to submit a single reconstruction seed for all children as they share a common seed. The lineage tracker has an idea of what reconstruction plans map to a single seed, and uses this information to decide which blocks to prune</td>
      <td>All blocks consumed exactly once, and number of seed re-submissions (i.e. ListFiles) tasks is kept minimal because of batching</td>
    </tr>
    <tr>
      <td>4c.</td>
      <td>Children fail one after another, or a child fails while a sibling's reconstruction is in flight</td>
      <td>Each later failure opens its own plan, and despite the children all sharing a common seed, the seed is re-submitted for each one</td>
      <td>All blocks consumed exactly once, might be many wasteful seed/ ListFiles tasks, resulting in slower pipeline</td>
    </tr>
    <tr>
      <th colspan="4" align="left">Chaos</th>
    </tr>
    <tr>
      <td>5a.</td>
      <td>Several nodes die at once</td>
      <td>Every lost block opens its own plan. Plans may have separate/repeated seeds. Withheld outputs are charged to the operator's object-store budget and operators could be backpressured.</td>
      <td>Slow at high rates of failure. (e.g. 6x/12x rate of our chaos release tests)</td>
    </tr>
    <tr>
      <td>5b.</td>
      <td>Head node dies</td>
      <td>The driver dies with it: executor, lineage tracker, every in-flight reconstruction</td>
      <td>Recovery is a job restart. Lineage tracker will be able to recover from checkpoint (TODO)</td>
    </tr>
  </tbody>
</table>

## Compatibility, Deprecation, and Migration Plan

- **Off by default.** `RAY_DATA_RECONSTRUCTION` defaults to `OFF`. With the flag off, behavior is unchanged: Core lineage reconstruction remains the fault tolerance mechanism for Ray Data.
- **Mutual exclusion with Core lineage pinning.** Enabling `RAY_DATA_RECONSTRUCTION` must also disable lineage pinning for Ray Data's objects. Running both mechanisms simultaneously would result in duplicated reconstruction of the same objects. The blocking issue is that lineage pinning is currently a cluster-wide environment variable; we need scoping so that non-Data workloads in the same cluster keep Core reconstruction. Until that scoping exists, enabling Ray Data reconstruction in a mixed cluster changes fault tolerance behavior for non-Data workloads, which is not acceptable for GA.
- **New public API.** `block_consumed` is purely additive. Users who never call it keep today's behavior, except that lineage for blocks whose refs outlive the pipeline cannot be garbage collected — the executor stays pinned. This is a memory-footprint consideration, not a correctness one, and should be documented.
- **Guarantee changes.** The mechanism is at-least-once and assumes deterministic UDFs — the same guarantees Core lineage reconstruction provides today, so no user-visible weakening. Output order is not preserved under failure.
- **No deprecation.** Nothing is deprecated by this REP. If Ray Data reconstruction becomes the default in a future release, deprecating Core lineage reconstruction *for Data workloads* would be proposed separately, after parity is demonstrated.

## Test Plan and Acceptance Criteria

### Unit tests

- Lineage graph construction: node/edge creation on task submission for fan-in, fan-out, and linear topologies; correct grouping of arguments by producer.
- Plan construction: correct set of seed tasks and correct downstream-task → output-index mappings for a failed task in each topology; no unrelated tasks included on fan-out.
- Resubmission: outputs needed by the plan are routed to downstream tasks; outputs not needed are dropped immediately.
- Failure mid-reconstruction: re-running the existing plan from the failed node, with no new plan created.
- Garbage collection: graph nodes removed on lineage completion; `clear_block_lineage` and `block_consumed` correctly resolve blocks and drop state; no leaked nodes after a full pipeline run.
- Data Task ID stability across reconstruction attempts.
- Admission control: reconstruction blocks never bundle with fresh blocks; a reconstruction task receives exactly the inputs its plan specifies.

### Integration and chaos tests

- **Node loss under load**: kill worker nodes at varying rates during a multi-stage pipeline (fan-in and fan-out topologies) and assert job completion and output correctness.
- **Batch node loss**: simultaneous loss of a large fraction of nodes, including losses that trigger overlapping reconstruction plans sharing upstream lineage.
- **Spot preemption simulation**: long-running pipeline on simulated spot instances.
- **Actor pool recovery**: verify a failed actor task is reconstructed on a *different* actor in the same pool, rather than waiting for the original actor to come back.
- **Sink and iterator paths**: `write_datasink` and `streaming_split` (Train dispatch loop) both survive node loss and produce correct output.
- **Resource stability**: assert object store usage and spilling stay within a bounded envelope of the failure-free baseline throughout the failure window.

### Performance tests

- Happy-path regression: throughput and memory overhead of lineage tracking with no failures, versus `RAY_DATA_RECONSTRUCTION=OFF`.
- Recovery time and total resource consumption versus Core lineage reconstruction on the same failure scenarios.
- Driver-side memory growth of the lineage graph on long-running pipelines with many tasks.

### Acceptance criteria

1. **Guaranteed completion**: pipelines complete successfully under repeated and batched node loss in the chaos suite.
2. **Stable resource utilization**: no excessive spilling during failure — object store usage stays within the bounded envelope defined above.
3. **Feature parity**: every fault tolerance scenario covered by Core lineage reconstruction for Ray Data today is covered by the new mechanism, demonstrated by the chaos suite.
4. **Observability**: metrics and logs expose reconstruction progress at the application level — tasks pending reconstruction, active plans, reconstruction task counts per operator — plus alerting on pathological FT behavior and a debugging runbook for on-call.
5. **Documentation**: user-facing docs for `RAY_DATA_RECONSTRUCTION`, `block_consumed`, and the guarantees (at-least-once, deterministic UDF assumption, no order preservation under failure).
6. **No happy-path regression**: throughput overhead of lineage tracking with the flag on and no failures is within an agreed threshold of baseline.

## Follow-on Work

1. **Core-side scoping of lineage pinning** — a job-level (or similar) configuration to disable Core lineage reconstruction only for Ray Data objects. Required before the flag can be enabled by default.
2. **Batching reconstruction triggers** — merge simultaneous reconstruction attempts that share upstream lineage, so the shared lineage is reconstructed once. Addresses the primary drawback of the plan-based design.
3. **Data–Train interface for T2A/T2B** — reconstruct exactly the blocks lost by a dead Train worker, and avoid reconstruction entirely when the producing Data workers are still alive.
4. **Broader consumption-interface support** — first-class support for `take`/`take_batch`, aggregations, `materialize`, and `to_random_access_dataset` beyond the initially supported write-to-sink and `streaming_split`.
5. **Reconstruction-specific observability** — recover the per-task progress visibility lost by sharing operator queues between fresh and reconstruction work.
6. **Core/Data hybrid design** — revisit Option 3 (see Appendix) for a more elegant division of responsibility once the Data-side implementation is in production.
7. **Stable-store evaluation** — benchmark checkpointing/WAL approaches against reconstruction for completeness.

## Appendix

### Alternative designs considered

#### Option 1: Resubmission from root

The simplest approach: each task records the *root task* it descends from, where a root task is the upstream-most task on the dependency chain that takes no dependencies.

When a task fails, the streaming executor on the driver detects the failure via Core reporting `ObjectLostError` or a variant of `TaskError`, looks up the root task the failed task depends on, and queues that root for resubmission.

**Pros**

- Resubmitted tasks enter the `StreamingExecutor`'s queue and are backpressured like normal tasks — no Core/Data split brain.
- Resubmitted tasks are new tasks to Core, so they are not bound to a specific actor. With actor pools, the task can run on any actor in the pool.
- Extremely simple to implement and maintain.
- Very low per-task driver overhead — essentially a map from each task to its root task.

**Cons**

- Only works for workloads with no fan-out. A fan-out task causes unrelated downstream tasks to be resubmitted as well.
- Still needs lineage tracking for the per-task mappings to be garbage collected correctly.

#### Option 2: Three-color graph reconstruction (rejected)

An attempt to fix the duplicated-shared-lineage problem in the plan-based design by giving each node in the same graph one of three states — `PENDING_RECONSTRUCTION`, `EXECUTING`, `COMPLETE` — instead of tracking plans.

Normal execution builds the same graph, plus: mark a node `EXECUTING` on submission and `COMPLETE` on completion. On failure, instead of building a plan, color the graph:

1. Transition the current node (starting with the failed task) to `PENDING_RECONSTRUCTION`.
2. Walk up to each parent.
3. Repeat until reaching a seed task marked `PENDING_RECONSTRUCTION`, or a parent that is already `PENDING_RECONSTRUCTION`.

Then resubmit the pending seed tasks (if no new pending seed tasks were added, no resubmission is needed): transition a task to `EXECUTING` when resubmitted; when a resubmitted task produces an output, resubmit any `PENDING_RECONSTRUCTION` child that needs it once its arguments are resolved, and otherwise drop the output.

Because the walk stops at nodes already marked `PENDING_RECONSTRUCTION`, simultaneous failures sharing upstream lineage do not duplicate that work. No special handling is needed for mid-reconstruction failure, since there is no notion of a reconstruction attempt. Metadata is lighter than plans and scales with the number of nodes rather than the number of attempts.

**Why we rejected it:** the design does not actually hold up. When reconstruction reaches a task that is *mid-execution*, the reconstruction may produce extra output — strictly worse behavior than Core's lineage reconstruction — and reaching an `EXECUTING` node that is itself part of a reconstruction still results in double work. Fixing this requires tracking state per *object* rather than per task, which is substantially more complex than the plan-based approach.

#### Option 3: Tracking via Ray Core (unexplored)

There is design space for a hybrid where Core tracks some lineage information (or simply exposes a better resubmission API) so Data does less work and we are not reimplementing lineage reconstruction. One idea considered and abandoned: have Core invoke a callback into Data whenever it triggers reconstruction, so Data can record the resource consumption while Core performs the reconstruction. This is unsatisfying because there are still two underlying scheduling mechanisms.

This space is worth revisiting. The position of this REP is that the Data-side reconstruction implementation is a P0 that immediately improves Ray Data fault tolerance, and a more elegant Core/Data split can be designed later.
