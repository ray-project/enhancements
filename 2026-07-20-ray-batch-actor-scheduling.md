# Summary

### General Motivation

Ray currently processes actor creation requests one by one. For workloads that spawn hundreds or thousands of actors (e.g., large-scale RL), this sequential creation can become a bottleneck because each actor registration requires a separate RPC call to the GCS server, and the scheduling of actors may be suboptimal when evaluated on a per-actor basis.

This REP introduces a `ray.batch()` API, which provides a native context manager for scheduling Ray actors in batches. When actors are created inside the context block, Ray buffers the actor creation requests and processes them in a single RPC to the GCS server. By batching actor creation requests, we introduce opportunities to reduce scheduling overhead and optimize placement and resource allocation.


### Should this change be within `ray` or outside?

Within Ray.

## Stewardship

### Required Reviewers

@MengjinYan
@Yicheng-Lu-llll

### Shepherd of the Proposal (should be a senior committer)

@edoakes

## Design and Architecture

### API

The proposal introduces a new Ray Core API to schedule actors in batches using a context manager. See the example below:

```python
import ray

# dummy class
@ray.remote
class MyActor:
    def __init__(self):
        pass

ray.init()

# create 1000 actors in a batch
with ray.batch():
    actors = [MyActor.remote() for _ in range(1000)]
```

### Implementation Details

Under the hood, the batch context manager will signal the Core Worker to buffer actor creation requests instead of sending them to the GCS immediately. This allows the actor scheduler to optimize actor placement for co-location and locality, reducing overhead and improving overall scheduling throughput.

To achieve this, we extend the actor creation workflow as follows:

* **ray.batch() API**: The `ray.batch()` context manager signals to the Core Worker (e.g. Cython CoreWorkerProcess.GetCoreWorker().EnterActorBatch()) to enter batching mode for the current thread. Upon exit, it calls ExitActorBatch().
* **Core Worker Actor Buffering**: Inside CoreWorker::CreateActor, if "batch mode" is enabled, the TaskSpecification for actors are pushed into a thread local buffer.
    * ActorIDs are generated synchronously (as usual via the worker context deterministic generator) and returned to Python immediately, so the user has valid ActorHandles.
    * When ExitActorBatch() is called, it flushes the buffer by invoking an RPC to the GCS server asynchronously to register all the actors in the buffer in one atomic operation (see RegisterActorBatchRequest below).
* **GCS RPC Extension (RegisterActorBatch)**: A new RPC RegisterActorBatchRequest will be added to gcs_service.proto.
    * GcsActorManager implements HandleRegisterActorBatch, which iterates through the batched tasks and invokes RegisterActor for each task locally in the GCS.
    * Once all tasks are successfully registered in the backend storage, a single RPC reply is returned to the Core Worker.
* **Submission Execution**: After the GCS responds to the RegisterActorBatch RPC, the Core Worker loops over the batch and calls actor_task_submitter_->SubmitActorCreationTask(task_spec) to push the tasks to the placement group / scheduling queue.

## Compatibility, Deprecation, and Migration Plan

This change introduces a new API (`ray.batch()`) and will not impact behavior of any
existing Ray API. No changes will be made to Ray's default actor scheduling.

## Test Plan and Acceptance Criteria

* Compare the latency and throughput of creating 1,000 and 10,000 actors with and without `ray.batch()`. Acceptance criteria includes a significant reduction in GCS CPU utilization and lower end-to-end actor creation latency.
* Ensure actors function identically whether inside or outside `ray.batch()` context manager.
* Unit, integration, e2e tests for `ray.batch()`

## Alternatives

### Alternative 1: `MyActor.batch_remote(num, *args, **kwargs)`

Instead of a context manager, introduce a new remote actor method like `MyActor.batch_remote(num, *args, **kwargs)`. While this approach could work, a context manager
provides more flexibility for batching heterogeneous actors with different resources/arguments in the same batch. 

### Alternative 2: Auto-batching of Actors

Automatically buffer actor creation requests in the background for a short duration without an explicit API. This approach introduces non-deterministic latency to individual actor creations and introduces significant risk of breaking existing behavior for Ray actors. 
