# REP: Ray Data Unified ActorPoolStrategy

## Summary

Replace Ray Data's mixed task/actor execution model with one in which every *map* operator runs on a pool of *actors*, and replace operator-level backpressure with **per-actor** input/output budgets enforced by Ray Data .

The change is gated behind `RAY_DATA_ACTOR_ONLY_BACKEND=1` and is off by default for the initial release. When enabled, Ray Data:

- promotes every `TaskPoolStrategy` to an `ActorPoolStrategy` (all tasks execute on actors)
- bounds the outstanding **output bytes/blocks per actor** and the in-flight **input tasks/bytes per actor**, replacing the confusing operator-level backpressure knobs with a single policy;
- takes ownership of **how many** actors each operator gets and **which node** each actor lands on, in a single decision made by a new `OperatorSizer`;
- tracks every live block's owning actor, operator and node in a new `ResourceBank`, so object-store accounting is attributable at node granularity.



### General Motivation



#### Problem 1: object spilling is a property of node-level skew, but backpressure is enforced at operator granularity

Ray Data today decides whether to admit more work by comparing an **operator's object-store usage** against a **cluster-global** budget. That comparison does not account for where the objects physically live. This makes **heterogeneous clusters inherently unstable.** When the `object_store / core` ratio differs across node types (batch inference and training ingest jobs frequently use mixed GPU and CPU nodes), operator-level admission control oversubscribes the memory-poor nodes. Ray Core's task scheduler can pack tasks onto a node whose object store is near the spilling threshold, because no component owns the per-node view. For example, `heterogeneous_memory_batch_inference` release test spills 350GiB on the memory-poor nodes while cluster-wide object store usage stays well under its limit.

#### Problem 2: System Complexity

The existing design exposes unclear responsibilities between Ray Data and Ray Core. For example: Ray Data decides *how many* tasks to submit, whereas Ray Core decides *where* to place them. In fact, Ray Core is currently responsible for fault tolerance in Ray Data. Due to this design, Ray Core can bypass Ray Data’s backpressure implementation (via lineage reconstruction), resulting in surprising memory usage and complex reasoning. Other surprises include idle worker reuse across multiple tasks, which require [hacky workarounds](https://github.com/ray-project/ray/pull/63490). These system complexities can be avoided by allowing Ray Data to own the backpressure end to end

#### Problem 3: Performance gaps

Today Ray Data supports a mixed task and actor execution model. Each regular task pays scheduling overhead costs. Here's a worst-case example at a high-level: When a task is ready to run in the cluster:

1. Wait 1 scheduling loop cycle (100ms); This is the typical cadence at which Ray Data submits tasks
2. Transfer primary object's location to a secondary location (100ms)
3. Execute task

It’s worth noting that this issue can also be solved via tasks, but will require substantial Ray Core changes. Actors are an existing paradigm that already solve this issue out-of-box, while also addressing the first 2 problems.

### Proposal: Unified ActorPoolStrategy

While Ray Data can make an effort to address each of the problems incrementally, the time to triage, debug, benchmark, and fix would eventually outweigh the attempt to fundamentally redesign Ray Data. Therefore, we propose to make every operator an actor pool, which gives Ray Data four things it cannot get from a task-based model:

- **A stable, addressable unit of execution with a known location.** An actor's node is (for the most part) fixed for its lifetime, so "how many bytes does this operator have on node N" is cheaply available, and a per-actor budget mechanically implies a per-node budget: with at most `M` actors on a node, giving each actor `object_store_per_node / M` bounds the node.
- **Task pipelining.** Multiple tasks can be queued to one actor, so task submission overlaps task execution, and the pipeline is far less sensitive to scheduling-loop latency: Ray Core runs the next queued task without waiting for Ray Data's next tick.
- **No worker reuse surprises.** Idle Ray Core workers holding heap memory after a task finishes are a recurring source of node OOMs that Ray Data (without some hacky runtimeenv) can't control.
- **Stateful GPU operators.** Tasks alone cannot support them.

The tradeoff is that Ray Data must now own decisions Ray Core used to make: placement, downscaling, task→actor routing, and retry of a task whose actor died. This proposal takes that trade deliberately — Ray Data already owns *when* to scale, so splitting *where* across two systems produces the reconciliation problems described above.

### Should this change be within `ray` or outside?

This belongs in the main `ray` project. The changes are in Ray Data, and development lives under `python/ray/data/_internal/experimental/`.

## Stewardship



### Required Reviewers

- @goutamvenkat-anyscale
- @bveeramani
- @edoakes
- @richardliaw
- @akshay-anyscale



### Shepherd of the Proposal (should be a senior committer)

- @edoakes



## Design and Architecture



### Overview

Four changes, in dependency order:

1. **Remove tasks from the map path.** Any `TaskPoolStrategy` is promoted to an `ActorPoolStrategy` at planning time.
2. **Make backpressure per-actor.** Each actor has a bounded budget of outstanding output bytes/blocks and in-flight input tasks/bytes.
3. **Make sizing and placement one decision.** A new `OperatorSizer` decides both how many actors each operator gets and which node each one lands on.
4. **Track object-store usage per actor/operator/node.** A new `ResourceBank` follows every block from creation to consumption, including cross-node copies.

(3) and (4) exist because (1) and (2) are not viable without them: a per-actor budget is only meaningful if you know how many actors share a node, and only enforceable if you know which node each block is on.

Each pass of the streaming executor's scheduling loop does three things, in order:

1. **Settle what finished.** Ingest the outputs of tasks that completed this tick, and update the ledger: credit the producing actor's output budget, release the consuming actor's input budget, and record the block's node so cross-node copies are counted.
2. **Size and place.** Decide how many and where to place actors for each operator.
3. **Admit work.** Submit queued input to actors that still have input budget, preferring the least-loaded actor whose task inputs are local.



### Per-actor backpressure

Every actor carries two budgets:

- **Output budget** — the bytes and blocks of this actor's outputs that exist and have not yet been consumed by a downstream task. Includes objects still inside the streaming generator's prebuffer, which are in the object store but not yet visible to the executor.
- **Input budget** — the tasks and bytes submitted to this actor and not yet completed. Counts both node-local inputs and inputs that require a cross-node copy.

An operator refuses to pull more outputs from an actor at its output limit (*output backpressure*), and refuses to submit more tasks to an actor at its input limit (*input backpressure*). Those two rules replace the entire list of environment variables in the Motivation section.

#### Accounting rules

The subtle part is when an upstream actor's output budget is released, because whether a downstream task needs a *copy/duplicate* of the block depends on whether the two actors share a node.


| Event                       | Downstream actor is node-local                                        | Downstream actor is remote                                            |
| --------------------------- | --------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Upstream task yields output | upstream output usage **+=** size                                     | upstream output usage **+=** size                                     |
| Task submitted downstream   | downstream input usage **+=** size; upstream output usage **−=** size | downstream input usage **+=** size;                                   |
| Downstream task completes   | downstream input usage **−=** size                                    | downstream input usage **−=** size; upstream output usage **−=** size |


The local case releases the producer early precisely because there is no second copy: the bytes are already charged to the consumer's input budget, and charging them twice would let a slow consumer artificially starve a fast producer. The remote case holds the producer's charge until completion, which is what accounts for the secondary copy on the consumer's node.

Note: If Ray Core made it possible to eagerly remove the primary copy, we could eliminate the need for duplicate accounting.

#### Deriving the per-actor limits

The per-actor output budget is **derived from the node**, not a global constant. The calculation follows something like this:

```
budget = (node_object_store_bytes × OUTPUT_FRACTION) / actors_on_node
budget = clamp(budget, MIN_OUTPUT_LIMIT, MAX_OUTPUT_LIMIT)
target_max_block_size = budget × BLOCK_SIZE_RATIO
```

Take the most typical example of 4 GiB / core. Approximately 25% of the memory will be reserved for object store, leaving behind 1 GiB / core. To account for both inputs and outputs of object store fed into and used by actors respectively, about 50% of object store will be reserved for outputs, and the other 50% for inputs.

### Operator sizing and placement

Note: The following is a high-level description of the proposed implementation. Future implementation details are subject to change in the future. 

The `OperatorSizer` runs in two modes.

- **Warmup** gathers signals on how much object store each operator produces, and how fast each task runs.
- **Warm** scales actors up and down from queue sizes and in-flight tasks. Together they replace the `ResourceManager` and `AutoscalingActorPool` with a single decision maker.

In either mode, each tick runs three steps:

```python
how_many = sizer.scale_how_many(topology)   # per-op deltas
where    = sizer.scale_where(how_many)      # per-actor node assignments
sizer.scale(where)                          # create / kill
```

Sizing is denominated in CPU, GPU and memory only. Object store is deliberately *not* a sizing input — it is handled by the per-actor budgets above.

#### Warmup phase

The sizer first goes through a warmup phase to gather signal (that is, to avoid over-provisioning source operators, which hold all the input at the start). At a high-level, the sizer evenly splits the cluster resources among all eligible (`ActorPoolMapOperator`) operators.

Take a `read → map → write` pipeline on a cluster of 10 nodes × 8 CPUs, where every actor asks for 1 CPU, and warmup is allowed to commit 80% of the cluster:

```
cluster total                    80 CPUs
minus each operator's min_size   −3      (3 operators, 1 actor each)
                                 ────
                                 77 CPUs × 80% = 61 CPUs to share

split 3 ways                     ~20 CPUs per operator
at 1 CPU per actor               20 actors each
```

Notes:

- Warmup never downscales: once an actor starts, it stays until warmup completes.
- CPU and GPU are split independently. A GPU operator's share is drawn from the cluster's GPUs, not its CPUs, so a wide CPU stage cannot crowd out the GPU stage the pipeline depends on. An operator that asks for both is sized against its GPU share and its CPU footprint is simply reserved.

Warmup ends when the **critical operator** — the last non-sink operator — produces its first output.

#### Warm phase

The goal of this phase is to make sure utilization is high (actors are busy doing work)

The existing design's main feedback for scaling decisions is influenced by queue sizes. In a nutshell, the `ResourceManager` recomputes the input queue sizes per operator. If the input queue is large, upscale. If actors are idle for a period of time, and there is 0 input, downscale. Similar to the existing design, the warm phase reuses the concept of queue sizes to direct scaling decisions. Contrary to the existing design, the warm phase will take a more holistic approach by looking at the entire DAG.

**Upscale**. For an upscale decision, the sizer targets these operators:

1. Operators available input
2. Operators that don't overrun the downstream operator (for example, if the candidate operator's output queue is large enough to feed input into the downstream, then upscaling would not improve the throughput)

If 1) and 2) are true, then the operator is *qualified* for upscale, but not *required* to downscale. There may be better candidates based on larger backlogged operators.

**Downscale**. For a downscale decision, the sizer targets idle, output-backpressured actors. Actors that are output-backpressured are holding execution units, preventing progress from being made.

#### Placement

`scale_where` assigns each new actor to a node.

**Upscale**, in priority order:

1. **Nodes that need this operator for locality.** A node qualifies when it already holds this operator's upstream or downstream actors, or its queued input bytes, but too few of its own actors to serve them. Nodes are ranked by how large that imbalance is, then by queued bytes per actor, then by node id for determinism.

1. **Everything else, spread proportionally** to each node's free slots.

Each actor is then created with `NodeAffinitySchedulingStrategy(soft=False)`.

**Downscale**, in priority order:

1. **On nodes an upscaling operator wants freed**, so the capacity goes somewhere useful. A node where this operator has pending or idle actors is ranked including queued bytes, since eviction there lands before the queue snapshot goes stale; a node offering only busy actors is ranked on structural imbalance alone, because by the time a busy actor drains, that snapshot no longer describes reality.
2. **Pending actors, then idle** — capacity frees immediately and no work is wasted.
3. **Busy actors last**, which drain rather than being killed.

Note that actors are not completely downscaled until their final outputs are consumed downstream. This makes object store accounting easily attributable

#### Scheduling constraints

Because Ray Data now controls actor placement, it must support these Ray Core scheduling constraints for parity:

- varying amounts of CPU, GPU, and memory
- label-based scheduling
- placement groups



#### Cluster autoscaling and the autoscaling coordinator

Node-level autoscaling reuses the existing cluster autoscaler. Unifying it with operator sizing into one decision is follow-up work.

### The ResourceBank

`ResourceBank` is a per-dataset ledger that models actors, tasks, operators and nodes, so object-store usage can be queried at any of those granularities. It is the mechanism behind both the backpressure accounting table and the per-node metrics.

```python
    class ResourceBank:
        stats: ClusterStats
        live_block_refs: Dict[ObjectRef[Block], UniqueBlockMetadata]
        _lock: threading.RLock   # executor thread + consumer thread (iter_batches)
```

The lock is reentrant because Ray Data runs a scheduling thread and a consumer thread, and getters call other getters.


| Callback                                          | When                                                                                                 |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `on_maybe_register_actor` / `on_deregister_actor` | actor created / removed; idempotent; records the actor's node and follows relocation across restarts |
| `on_task_submitted`                               | task → actor mapping; where cross-node-copy tracking begins                                          |
| `on_new_output`                                   | a running task yields a block, or `ray.put` for in-memory data                                       |
| `on_task_completed`                               | decrement in-flight                                                                                  |
| `on_output_consumed`                              | a downstream task that used this block completed; the block leaves the ledger                        |


Queries: `live_object_store(actor | node | operator)` broken down by inputs/outputs/local/remote and by pulled vs. prebuffered.

Because actor locations and block locations are both known, the bank can tell in advance whether submitting a given bundle to a given actor will force a secondary copy — which is what makes the local/remote accounting split, and locality-aware routing, possible.

Prebuffered objects are the one estimated quantity: their sizes are not observable until they leave the generator, so a default (or the operator's observed average block size) is used.

### Actor pool changes

A new `_ExperimentalActorPool` adds a byte-denominated input limit alongside the task-count limit, and defines the "least-loaded" actor as fewest tasks in flight, tie-broken by oldest last-submission timestamp.

Actor lifecycle:


| State      | Meaning                                                                                                                                             |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PENDING`  | created, not yet accepting tasks. Expires after a generous timeout, which reclaims actors wedged on a failed restart or a stuck runtime-env install |
| `RUNNING`  | in the scheduling heaps; eligible for task submission                                                                                               |
| `DRAINING` | marked for termination; no new tasks. Warns if it holds node capacity for too long                                                                  |
| `RELEASED` | in-flight tasks done and output blocks consumed; capacity returned                                                                                  |




### Fault Tolerance

Ray Core lineage reconstruction and retry tasks are currently opaque to the Ray Data scheduler. Therefore this design depends on future work that moves fault tolerance into Ray Data, making it easier to reason about node deaths and actor restarts. 

### Scheduling loop changes

The existing loop asks `op.can_add_input()` *before* the operator has seen the bundle, then adds it. Deciding admission without seeing the input is workable for tasks but wrong for actors, where the routing decision (which actor, and therefore whether a cross-node copy is needed) depends on the bundle. It has also produced skipped bundles and near-livelocks under spot preemption.

The actor-only loop decouples the two:

```python
def _scheduling_loop_step(self, topology):
    self._resource_bank.set_node_view(reporter.get_reserved_resources_by_node())
    self._resource_bank.drain_consumed_blocks()

    process_completed_tasks(topology, ...)     # always ingest outputs; OBP handled here
    detect_if_idle(...)                        # deadlock escape hatch (adaptive interval)

    self._sizer.observe(topology)
    how_many = self._sizer.scale_how_many(topology)
    where    = self._sizer.scale_where(how_many)
    self._sizer.scale(where)

    self._resource_bank.drain_consumed_blocks()
    self._launch_tasks()                       # admission, now that inputs are visible

    self._actor_only_metrics.maybe_update(...)
    self._cluster_autoscaler.try_trigger_scaling()

def _launch_tasks(self):
    for op in self._topology:
        if isinstance(op, ExperimentalAPMO):
            while op.can_submit_task():        # per-actor input budget
                op.launch_task()               # pick the least-loaded actor, submit
```

`select_operators_to_run` is gone: every operator runs whenever it has input capacity. The operator interface gains `can_submit_task()` and `launch_task()` alongside the existing `can_add_input()`. Decoupling input admission from task submission is worth backporting to the task+actor backend independently of this proposal.

### Operator fusion

Actor→actor fusion is enabled (the legacy rules allow task→task and task→actor but not actor→actor), subject to:

- identical compute strategies on both sides, so the fused operator has an unambiguous pool configuration;
- **no async UDF on either side.** Async+sync would conflate two different concurrency models in one pool, so the async stage stops honoring its own `max_concurrency`. Async+async deadlocks before the first UDF call, because the sync↔async bridge generator assumes its consumer is the task's main thread rather than the actor's shared event loop;
- compatible `ray_remote_args` (`max_concurrency`, `num_cpus`, … must match);
- read operators are not fused.



### Memory reservations

If the user does not set `memory`, Ray Data reserves none per actor. If `DataContext.get_current().default_map_logical_memory_enabled = True`, actors will be scheduled with that memory limit. Because actors cannot change their memory reservation after placement, future work will involve incorporating observed USS into the actor's memory limits.

### Ray Core dependencies

Ray Data's existing `_generator_backpressure_num_objects` is **per task**. With `max_concurrency=k`, an actor can hold `k ×` the intended prebuffer, over-subscribing the per-actor output limit. Two Ray Core integration changes are required:

1. `_actor_generator_backpressure_num_objects` — an actor-scoped generator backpressure limit on the number of prebuffered objects, fixed regardless of how many tasks the actor runs concurrently.
2. `_num_objects_per_yield` — grouped yields. Ray Data tasks yield a block and then its metadata as two separate objects. Under `max_concurrency >> 1` an actor can emit all its blocks before any metadata, and the executor cannot make progress until a later tick, which is a livelock in the general case. Yielding the pair atomically (`_num_objects_per_yield=2`) fixes it with minimal Data-side change.



### API Design



#### Enabling

```bash
export RAY_DATA_ACTOR_ONLY_BACKEND=1
```

Using an actor-only-specific API while the backend is off raises, rather than silently no-op'ing.

```
ActorPoolStrategy
```

Two new arguments:

```python
    ActorPoolStrategy(
        min_size=None,
        max_size=None,
        initial_size=None,
        max_tasks_in_flight_per_actor=None,   # existing: in-flight task count per actor
        max_num_output_bytes_per_actor=None,  # NEW: outstanding output bytes per actor
        max_num_outputs_per_actor=None,       # NEW: outstanding output blocks per actor
        enable_true_multi_threading=None,
    )
```

`None` means "derive from the node", per the formula above. Setting them too high reintroduces spilling; too low forces the deadlock escape hatch to fire, which also reintroduces spilling risk. The intended posture is to leave them unset.

Note that input bytes limit is not exposed. The reason is that `max_tasks_in_flight_per_actor` is another knob that controls how many input bytes are submitted to an actor. During experimentation, we found that deprecating `max_tasks_in_flight_per_actor` leads to some regressions, so this parameter will stay for the time being.

#### Automatic task→actor promotion

A developer API called `maybe_promote_compute_strategy` converts any `TaskPoolStrategy` to an equivalent `ActorPoolStrategy` when the backend is on — fixed-size task pools to fixed-size actor pools, unsized to autoscaling.

**ray_remote_args**


| Argument                              | Status                                                              |
| ------------------------------------- | ------------------------------------------------------------------- |
| `num_cpus`, `num_gpus`, `memory`      | supported, any value                                                |
| `label_selector`                      | supported (actors placed untargeted; ordered first)                 |
| `PlacementGroupSchedulingStrategy`    | supported (pool capped by bundle capacity)                          |
| `NodeAffinitySchedulingStrategy`      | supported (local-scheme reads)                                      |
| `ray_remote_args_fn`                  | supported **only** if it assigns a placement group and nothing else |
| `resources` (custom)                  | rejected — capacity model tracks CPU/GPU/memory only                |
| any other `scheduling_strategy`       | rejected — the sizer owns placement                                 |
| `placement_group_capture_child_tasks` | rejected                                                            |


Rejections raise `NotImplementedError` at bootstrap, naming the operator and the offending argument.

#### Startup wait

When the actor-only backend is on, Ray Data waits a bounded time for the minimum number of actors:

```
time_to_wait_for_min_actors = min(BASE + PER_ACTOR × num_pending, MAX)
```

The wait aborts early if **no** actor becomes ready within a stall window. This can be configured through `RAY_DATA_DEFAULT_WAIT_FOR_MIN_ACTORS_S`.

#### Tunables

Every policy described above — budget derivation, sizer cadence and step sizes, warmup reservation, ordering, constraint awareness, the rebalancing guards, read-actor concurrency, teardown mode, fusion — is backed by a `RAY_DATA_*` environment variable declared in `ray.data._internal.execution.execution_flags`. These are operator and development knobs, not user-facing API.

### Observability

Per-node and per-operator visibility is a requirement to observe any node-level skews. The whole argument for this design is that node-level skew is invisible today. We plan to have metrics on:

- object store per node
- number of actors per operator
- input locality per operator
- scaling decisions
- actor lifecycle per operator
- distribution of actor output limit

Per-actor metrics are deliberately *not* exported: cardinality at tens of thousands of actors is prohibitive. Per-actor state is reachable through logs, which carry each sizing decision with its inputs and outcome, and warn on scheduling-loop lag.

### Examples

Enabling the backend requires no code change:
```bash
    RAY_DATA_ACTOR_ONLY_BACKEND=1 python my_batch_inference_job.py
```
```python
    import ray

    ds = ray.data.read_parquet("s3://bucket/input/")
    # Runs on a task pool today; promoted to an autoscaling actor pool automatically.
    ds = ds.map_batches(preprocess, batch_size=256)
    # Already an actor pool; now sized and placed by the OperatorSizer.
    ds = ds.map_batches(
        Classifier,
        batch_size=64,
        num_gpus=1,
        concurrency=(2, 32),
    )
    ds.write_parquet("s3://bucket/output/")
```

Overriding the per-actor budgets (not recommended; shown for completeness):

```python
    from ray.data import ActorPoolStrategy

    ds.map_batches(
        LargeFrameUDF,
        compute=ActorPoolStrategy(
            min_size=2,
            max_size=32,
            max_num_output_bytes_per_actor=1024 * 1024 * 1024,  # 1 GiB of outstanding output
            max_tasks_in_flight_per_actor=4,
        ),
    )
```

Placement-group workloads (e.g. vLLM's `ray` backend) work through a `ray_remote_args_fn` that assigns only a placement group:

```python
    ds.map_batches(
        VLLMPredictor,
        num_gpus=0,                              # GPUs come from the PG bundles
        ray_remote_args_fn=lambda: {"scheduling_strategy": make_pg_strategy()},
        concurrency=8,
    )
```

An unsupported argument fails loudly at bootstrap rather than mis-placing actors:

```python
    ds.map_batches(fn, ray_remote_args={"resources": {"custom_accelerator": 1}})
    # NotImplementedError: The Ray Data operator sizer does not support MapBatches(fn):
    # it specifies custom resources (the capacity model only tracks cpu/gpu/memory).
```



## Compatibility, Deprecation, and Migration Plan



### Backwards compatibility

The backend is **off by default**. With `RAY_DATA_ACTOR_ONLY_BACKEND=0` (the default), planning, execution, fusion, sizing and backpressure are byte-for-byte the legacy path: the actor-only executor, operator, sizer and fusion rule live in `_internal/experimental/` and are installed into the ruleset only when the flag is on.

No public API is removed or changed in meaning. `ActorPoolStrategy` gains three optional keyword arguments that default to `None`.

### What changes when a user opts in


| Area                                | Behavior                                                                                                                                                                            |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Compute strategy                    | `TaskPoolStrategy` is silently promoted to `ActorPoolStrategy`                                                                                                                      |
| Backpressure                        | operator-level policies are inert; per-actor budgets govern. The `RAY_DATA_DOWNSTREAM_*`, `RAY_DATA_OP_RESERVATION_RATIO`, `RAY_DATA_CONCURRENCY_CAP_*` knobs stop having an effect |
| Placement                           | Ray Data places actors via `NodeAffinity(soft=False)`; a user `scheduling_strategy` other than NodeAffinity or a PG raises                                                          |
| Custom `resources`                  | raises `NotImplementedError`                                                                                                                                                        |
| `preserve_order=True`               | raises `ValueError`                                                                                                                                                                 |
| Shuffles / all-to-all / joins / zip | `AllToAllOperator`, `HashShuffleOperator`, `HashAggregateOperator`, `JoinOperator`, `ShuffleMapOp`, `ShuffleReduceOp`, `ZipOperator` raise at bootstrap                             |
| Fusion                              | actor→actor fusion becomes possible; read operators are never fused                                                                                                                 |
| Startup                             | `wait_for_min_actors_s` becomes size-scaled with a stall abort, unless set explicitly                                                                                               |


Every unsupported case raises at bootstrap with the operator name and the reason. There is no silent fallback: a job that would be mis-executed fails to start instead.

### Scope of the initial release

**Supported.** Batch inference; training ingest without `preserve_order`; heterogeneous clusters; preemptible/spot clusters; cluster autoscaling; `ActorPoolMapOperator`, `Limit`, `Union`, `OutputSplitter`; placement groups and label selectors; datasource v2.

**Not supported.** `preserve_order=True`; all-to-all and shuffle operators (hash shuffle, hash aggregate, join, zip); `TaskPoolMapOperator`; custom resources.

### Migration path

1. **Now** — off by default; opt in with the flag. Legacy path untouched.
2. **Next** — default on for supported workload shapes, with the flag as the escape hatch. Requires the release-test bar in the next section to be met.
3. **Later** — once shuffles, `preserve_order` and fault tolerance land, remove `ResourceManager`, `AutoscalingActorPool`, and the operator-level backpressure policies, and fold `OpRuntimeMetrics` into `ResourceBank`. `RateBasedClusterAutoscaler` is retained.

Deprecation of the 10 backpressure environment variables happens at step 3, with the usual deprecation-warning cycle. They are already inert under the new backend at step 1, so a user who has tuned them will see them stop mattering the moment they opt in — this must be called out in the release notes and in the migration guide.

## Test Plan and Acceptance Criteria



### Unit Tests

A dedicated `python/ray/data/tests/actor_only/` suite covers:

- **Accounting**: the local vs. remote budget-release table, prebuffer size estimation, actor register/deregister/relocation, and thread safety against the consumer thread.
- **Actor pool**: the state machine (pending → running → draining → released), least-loaded routing and its tie-break, pending expiry, and drain reclaim.
- **Sizing and placement**: warmup ceiling math, the warm decision table and its hysteresis, tick-local grant accounting, locality rank, the collocate and spread phases, victim selection, and placement under label selectors and placement groups.
- **Rebalancing**: flow-controlled growth grants, donor capacity assessment, victim ordering, and refusal accounting.
- **Liveness**: every deadlock escape-hatch trigger, and that it fires at most once per call.
- **Gating**: task→actor promotion, actor→actor fusion rules and their exclusions, and that every unsupported configuration raises with an actionable message.

Existing streaming-executor, actor-pool, fusion, autoscaler and coordinator suites are extended so the legacy path is proven unchanged.

### Integration & E2E Tests

Release tests are run **paired** — the same workload on the legacy backend and on actor-only — so every claim below is a comparison, not an absolute.

- **Placement A/B.** `map_batches_placement_`* and `image_embedding_jsonl_subset_*` each run four arms: task-pool baseline, actor-pool baseline, actor-only with Ray Core `SPREAD`, actor-only with locality placement. This isolates the placement strategy's contribution from the backend's.
- **Heterogeneous clusters.** `heterogeneous_memory_batch_inference` plus a multi-tenancy variant — the case operator-level backpressure cannot express.
- **Scheduling constraints.** `dataset_constraint_label_scheduling` (parameterized) and `dataset_constraint_pg_scheduling`.
- **Cluster autoscaling.** Large-cluster and subset-cluster arms of `image_embedding_from_jsonl`, including a fake-GPU variant that reproduces the shape cheaply.
- **Training ingest.** `training_ingest_benchmark` parameterized by backend.
- **Chaos.** Two actor-only chaos tests — immediate actor kill and graceful drain — each run as 3 trials, plus spot-preemption coverage.

New tests are still needed for: row expansion, fast producers, deliberately bad initial sizing, uneven UDF durations, fractional resource requests, and pipelines mixing actor and non-actor operators.

### Acceptance Criteria

1. **No spilling** on the existing Data release-test suite plus the new suite, for batch inference, training ingest without `preserve_order`, and cluster autoscaling.
2. **Limited performance regression** against the legacy backend on the same suite, arm for arm. Known-expected exceptions must be enumerated and justified — in particular synthetic/in-memory datasets, where read operators are never fused.
3. **No deadlocks** under adversarial configuration: bad `concurrency`, tiny or unbounded `target_max_block_size`, and initial sizing chosen to be wrong. Where the escape hatch fires, it must be visible in the `Output Limit Upgrades` metric.
4. **Scalability** at least at parity with the current backend — tens of thousands of actors — with evidence that scheduling-loop cost is linear in cluster size, and a documented path to another order of magnitude.
5. **Fault tolerance**: chaos tests (kill and drain) pass across trials with no leaked capacity, no wedged pendings, and no bootlooping actors.
6. **Every unsupported configuration raises at bootstrap** with an actionable message. Verified by `test_disabled_configs.py`.
7. **Observability shipped**: the actor-only Grafana panels populate on a real run, and the per-node object-store breakdown is sufficient to attribute a spill without a heap dump.
8. **Docs**: a migration guide covering the promotion of tasks to actors, the inert backpressure knobs, the unsupported surface, and how to read the new dashboards.

## Early Results

To test our prototype, we considered a couple real-world batch-inference problems.

Consider creating embeddings from 10 TiBs of raw images using google’s vision transformer: `google/vit-base-patch16-224`

```python
Read(CPU) -> Preprocess(CPU) -> create_image_embedding(GPU) -> Write(CPU)
```

We ran this pipeline on a cluster with 100 r6a.8xlarge, 40 g5.4xlarge. The existing design undergoes \~160GiB of spilling, and runs in \~700 seconds. The prototype design undergoes 0 spilling, and runs in \~600 seconds.

Similar trends in spilling and runtime were observed for other large-scale batch-inference jobs.

## Known Limitations and Risks

**Object store.**

- Actors that die before their outputs are consumed leave dangling outputs charged to a pool that no longer exists. Assumed rare; reported by metric when it happens.
- Every escape-hatch upgrade raises an actor's output limit and therefore raises spilling risk. This is a deliberate liveness-over-stability trade, and it is instrumented.
- The terminal operator's guarantee depends on the consumer dropping references before requesting more. Even a well-behaved consumer's own buffers (split coordinator, Train worker prefetch) add object-store pressure Ray Data does not control.
- Very large rows (video, large images) cannot be split, so a single row can exceed a per-actor budget. The fallbacks are input backpressure and downscaling that operator — neither of which is a guarantee. A per-actor byte budget may not be the right primitive for these workloads.
- `target_max_block_size=None` with blocks larger than the derived budget will spill. Forcing repartition on users who explicitly disabled it is not obviously correct.
- Because the `_actor_generator_backpressure_num_objects` is in terms of objects, operators that produce tiny outputs like GPU inference can be bottlenecked on producing tiny outputs fast enough to keep up with the scheduler's demand. In the future, we plan on removing a prebuffer to making everything in terms of bytes.

**Cluster shape.**

- Because actors outlive tasks, a small node must host at least one actor per unfused operator: an 8-CPU single-node cluster cannot run more than 8 unfused 1-CPU operators.
- `num_cpus << 1` packs many actors per node, shrinking every budget on it.
- Sizing count and placement are computed separately, so a shape like "5 CPUs per actor on 8-CPU nodes" yields a count that cannot be fully placed (800 CPUs ⇒ 160 actors, but only 100 nodes × 1 actor fits). The next tick reconciles, but the transient is real. This failure mode exists in the current backend too; here it is at least detectable.

**Ordering and pipelining.**

- `preserve_order=True` is rejected. Beyond the ordering constraint itself, an actor is not released until its outputs are consumed, so ordered pipelines cannot pipeline scale-down, and an actor may hold CPU/GPU while waiting on a strictly-ordered consumer.
- Actors may execute tasks out of submission order (`ALLOW_OUT_OF_ORDER_EXECUTION`), and streaming-generator acknowledgments may be reordered even at `max_concurrency=1`. The interaction of out-of-order completion with a per-actor output budget is a suspected deadlock source and needs dedicated testing.
- Async actors are the least-exercised path.

**Performance.**

- Read operators are never fused, so synthetic and in-memory datasets may regress.
- Actor startup is small but non-zero, which shows up on short pipelines.
- A large default `batch_size` against large rows (1 MiB/row at `batch_size=1024`) pipelines poorly against the input budget.
- We also lose the ability to time multiplex between multiple actors. Once an actor task is submitted, it cannot be resubmitted to another actor without substantial overhead. One common use case would be to load balance work across a pool of actors. If this does become an issue, Ray Core can implement an actor pool

**Placement and scale.**

- Placement runs on the scheduling loop and has known bottlenecks; it may need to move off the loop. On large clusters the new scheduling loop can reach 5-10 seconds.
- The sizer depends on an accurate view of reserved resources; any drift leaves actors pending.

**Fault tolerance.**

- With the new unified actor pool model, task routing is now handled by Ray data. Therefore, more work will be needed to make actor restarts and node deaths robust (for example, prioritizing submitting tasks to non-draining nodes, prioritizing consuming outputs of draining actors). The initial implementation may not be reliable without an implementation of fault tolerance that sits in the Ray Data layer.



## Follow-up Work

- **Shuffles.** Hash shuffle has been revamped to be actor-less; reconciling that with operator sizing is the main open question and the largest remaining gap.
- `preserve_order`**.** Potentially requires decoupling actor release from output consumption.
- **Fault tolerance.** Task retry onto a different actor; node-loss handling; revisiting `NodeAffinity` softness.
- **Unify cluster autoscaling with operator sizing.** They are two decision makers with correlated effects; the same argument that motivates merging sizing and placement applies here.
- **Merge** `OpRuntimeMetrics` **into** `ResourceBank`**.** Deferred until the backend is proven.
- **Ray Core "move" semantics**, which would collapse the local/remote accounting split into a single rule.
- **Per-node read-actor caps**, once NIC saturation thresholds are characterized per node type.
- **Moving the Ray Data actor pool into Ray Core.**
- Read + Write actor `max_concurrency` to saturate NIC bandwidth while using few CPUs
- **Experimenting with rate-based sizing**
- Removing the prebuffer to support actor limits solely in terms of bytes.
- Faster draining of actors by allowing them to downscale with unconsumed outputs and work stealing
- Allowing Ray Data to handle actor restarts. The scheduling should be explicit in ray data, and should allow actors to reschedule with observed memory usage.

