# Accelerated DAGs

Tl;dr: We propose changes needed to make Ray Core into an
accelerator-native execution substrate.

The two major goals are:

-   Task overheads in the tens of microseconds, compared to 1ms or more
    today
-   Support GPU-native communication, compared to CPU-only object store
    today

Either of these alone also has benefits:

-   Lower task overheads can be applied to other application use cases
    such as web serving, as long as the task control flow is
    relatively static and predictable.
-   GPU-native communication can make it easier to develop distributed
    ML applications

But together, we believe they can make Ray Core into an
accelerator-native execution substrate, allowing Ray Core to execute as
fast as SPMD programs but with much more flexibility.

## Required Reviewers
@ericl, @pcmoritz, @scv119

## Shepherd
@ericl


Background
==========

All popular execution frameworks (Distributed TensorFlow,
torch.distributed, etc.) for distributed ML today are based on a
"multi-controller" setup, aka
[SPMD](https://en.wikipedia.org/wiki/Single_program,_multiple_data).
In this setup, all worker processes run the same program. This has been
the status quo because: (a) it **minimizes system overhead** and (b) ML
apps often contain **all-to-all communication** steps for which SPMD is
well-suited.

<img style="background-color:white" src="2023-12-04-accelerated-dag-figures/image5.png">

*SPMD execution*


On the other hand, there has been recent evidence of the two major
downsides of SPMD:

-   **Flexibility:** It is difficult to express more complex
    applications that might involve multiple models, parallelism
    strategies, and/or dynamic control flow. See the [Pathways
    paper](https://arxiv.org/pdf/2203.12533.pdf) for more
    information.

-   **Fault tolerance:** The SPMD model requires all processes to run
    the same program. If one process goes down, everyone must restart
    leading to significant downtime. This is a pain point in
    large-scale training.

Ray uses a single-controller model, where a "driver" coordinates the
execution. It can also wrap multi-controller frameworks, as Ray Train
does, but its interface is primarily based on a single-controller model.

<img style="background-color:white" src="2023-12-04-accelerated-dag-figures/image4.png">

*Ray Train example: Ray Train is built on Ray's single-controller model, but the execution is still SPMD.*

Ray has the exact opposite properties as SPMD systems, prioritizing
flexibility and fault tolerance at the cost of:

-   **High control plane overheads:** Because tasks are dynamic (they
    can be spawned at any time on any process), Ray has to perform
    some bookkeeping for each individual task execution. This
    bookkeeping sometimes requires distributed protocols, such as for
    scheduling and garbage collection. This means that Ray developers
    need to make sure that each task has at least 10s of ms of
    computation, to reduce system overheads.

    -   Ray Train example: Coordination tasks need to be either
        relatively coarse-grained (e.g., contain many training steps)
        or asynchronous with training (e.g., metrics reporting).

-   **Generic CPU-based communication transport**: Ray's flexibility
    requires it to be highly general, so it implements a generic
    TCP-based communication transport. Adding native support for
    specialized communication transports such as GPU collective
    communication would be thorny.

    -   Ray Train example: Ray only provides generic communication, so
        GPU communication takes place out-of-band from Ray's
        perspective. This is not necessarily a problem; actually it's
        the recommended approach for accelerators. But the issue is
        that Ray doesn\'t really have any control over the
        communication then, so optimizations that Ray can do for CPU
        memory are not possible for GPU memory, like recovering
        quickly from communication failure or overlapping with
        compute.

<img style="background-color:white" src="2023-12-04-accelerated-dag-figures/image3.png">

*Vision for Ray Train in Ray 3.0: Ray Train is built on Ray's single-controller model, but with much finer-grained tasks and more control over the execution. The physical execution is as fast as SPMD.*

The "accelerated DAGs" effort aims to keep Ray's flexibility and fault
tolerance, but: (a) reduce control plane overheads, and (b) support
specialized communication transports. The basic ideas are to: (a) reuse
control plane decisions from past executions, (b) support
application-defined transports such as
[NCCL](https://developer.nvidia.com/nccl) and
[UCX](https://openucx.org/). Ultimately, the goal is to
make Ray into a native execution substrate for distributed ML
applications.

Architecture
============

## Compiled DAGs

***Goal:** Reduce control plane overhead in Ray Core task execution.*

Ray's control plane overheads come from the dynamic nature of the API.
Tasks can be spawned dynamically and they execute eagerly. Each time a
task is spawned, we cannot be sure what size outputs it will have, who
will depend on it, etc. This produces a number of overheads, including:

-   Allocating and ref-counting an ObjectRef
-   Putting, getting, and transferring a shared-memory object
-   Scheduling, dispatching, and finishing a task

<img style="background-color:white" src="2023-12-04-accelerated-dag-figures/image1.png">

*Diagram showing (some) of the synchronous steps that happen in today's Ray Core to execute a program like B.foo.remote(A.bar.remote()). Most of these communication edges take place over RPC/IPC. Control plane (driver, raylet) and data plane (actors, object store) operations are all interleaved with each other because of the dynamic execution model.*

Although some operations can be overlapped with other tasks (e.g.,
transferring arguments for one task while executing another),
fundamentally most of these operations need to be done *synchronously
with the task execution.* Overall these overheads add up to \~1ms of
execution overhead per task, even when execution is on the same node.

Essentially, the current model is "interpreted". But this isn't
necessary for cases where a similar DAG will get executed again. For a
given task, we can avoid most of the above steps and protocols as long
as we have information such as the dependent tasks and the max size of
the task's outputs.

***Key idea:*** *If we know that a certain DAG pattern will be repeated
in the future, we can "compile" a* *dataplane path for that DAG, thus
avoiding control plane overheads during execution. *

Because we no longer need to make control plane decisions in the
dataplane path, synchronization on a local node can be done via
shared-memory. Here is an overview of the per-task execution overheads
(note that there will be more overhead if significant serialization is
involved):

| **Before**                        | **After**                         |
|-----------------------------------|-----------------------------------|
| Many (\~10+) RPCs/IPCs, even more if inputs/outputs are remote | Shared-memory mutex for synchronization + ½ RPC if inputs/outputs are remote         |
| → **500us-1ms** for local + several RPCs/IPCs for remote | → **10s-100 of us** for local + ½ RPC for remote                    |


<img style="background-color:white" src="2023-12-04-accelerated-dag-figures/image2.png">

*Left: Instantiation. "Compile" a DAG before the first execution, by allocating buffers for task inputs/outputs and sending the task descriptions to the actor executors. Right: Execute the compiled dataplane. Communication edges can take place over shared memory when local.*

To accomplish this, we will extend the current [lazy DAG
API](https://docs.ray.io/en/latest/ray-core/ray-dag.html)
with a special "compiled" option. This API already allows DAG reuse but
currently carries the same execution overheads described above. The main
Ray Core APIs based on individual task and actor calls will not be
affected, but will be compatible with compiled DAGs, i.e. compiled DAGs'
outputs can be passed as arguments to normal tasks, and vice versa. In
the future, we may consider "just-in-time" compiling individual Ray
tasks.

To support compiled DAGs, we need to make a few assumptions:

-   The application can be expressed as a handful of
    [DAGs](https://docs.ray.io/en/latest/ray-core/ray-dag.html).

-   Tasks may dynamically call Ray tasks as usual, but the execution of
    these nested tasks will not be accelerated.

-   A DAG must be declared before it can be executed. The initial
    declaration may be higher latency than a single DAG execution.

    -   For now, dynamic control/data flow within a DAG will not be
        allowed. These could eventually be supported by splitting a
        DAG into sub-DAGs, and using the driver to coordinate the
        control/data flow between sub-DAGs.

    -   The max size of task outputs must be declared.

    -   A task output can only be read by the downstream DAG nodes,
        unless it is one of the final outputs of the DAG. (i.e. one
        cannot access intermediate outputs through the usual ObjectRef
        API).

### Fault tolerance

Initially, we will provide failure detection at the DAG level. In
particular, if any task in the DAG fails due to application exception or
worker death, we will propagate the error to the DAG outputs.

Currently, Ray Core also provides automatic task-level re-execution. At
the moment, we do not plan to support this, as it would likely require
extra overhead from tracking intermediate task outputs. However,
automatic DAG-level re-execution may be possible.

## Application-defined transports

***Goal:** Give the application greater control over the communication
method, but allow Ray Core to schedule the communication.* This could
allow us to keep current Ray Core system optimizations such as:

1.  Overlapping communication with task execution.
2.  Handle communication failures gracefully through retries.
3.  Control communication concurrency / resource usage.

Accelerated DAGs will ship with a default transport for objects passed
along the DAG's edges. This will use shared memory if the
sender/receiver are colocated, and shared memory/Ray Core for
cross-node.

To support specialized transports, we will extend the DAG API to allow
application-defined transports that can be called from either Python or
C++. The exact API is TBD, but here is a simple initial proposal based
on a Channel concept:

```python
class Transport:
  def __init__(self, sender: ray.ActorHandle, receiver: ray.ActorHandle):
    pass
  def send(self, send_meta: BufferMetadata):
    """Ideally async. Called by the sender"""
    pass
  def recv(self, recv_meta: BufferMetadata) -> Buffer:
    """Ideally async. Called by the receiver."""
    pass
```

When an application-defined transport is used, we will use the default
transport to synchronize between the sender and receiver, but we only
send the BufferMetadata in along the default transport. This will act as
a signal to begin the actual \`Transport.send\` and \`Transport.recv\`
of the application data.

Initially, our goal is to add a UCX-based transport, which supports
GPU-GPU and RDMA, among others. For GPU objects, we can also provide a
fallback transport, i.e. when a specialized GPU-GPU transport is not
available, the fallback transport copies the GPU object to the Ray
object store, then uses the default transport to transfer the object.

The other potential complexity in this work is supporting collective
communication, which can deadlock if not properly scheduled. During the
initial prototyping, we can continue to execute collective communication
out-of-band, as part of the application code. However, to realize the
long-term goals, we will introduce an API that captures collective
communication ops as part of the DAG. An example API:

```python
workers = [Worker.remote() for worker in workers]
# A task called on the CollectiveGroup will be gang-scheduled on each worker.
pool = CollectiveGroup(workers)
outputs = pool.allreduce.bind(inputs)
```

Workloads
=========

Possible validation workloads fall under two different categories:

**Common patterns in distributed inference and training:** These are
workloads that may be distributed but only use one type of parallelism
(e.g., tensor- vs pipeline-parallel) and only one model. They are
typically already supported by other multi-controller frameworks such as
DeepSpeed, Distributed TF/PyTorch, etc.

Goals:

-   Performance parity to validate system overheads
-   Validate API expressivity - we want to show that it is much simpler
    to implement these patterns with a single-controller interface
-   Increase the flexibility of (some) patterns. For example, pipeline
    parallelism is a well-known technique provided by Deepspeed, but
    Deepspeed only supports it for sequential
    modules
    ([link](https://www.deepspeed.ai/tutorials/pipeline/#getting-starting-with-pipeline-parallelism)),
    meaning that it can't be used for DAGs of modules.

**Advanced patterns:** These workloads may use multiple types of
parallelism, multiple models, and/or are more "dynamic" in some sense
(e.g., providing low down time after cluster size changes). In many
cases, there may be no existing popular implementation.

Goals:

-   Validate API expressivity
-   Validate integration with higher-level ML frameworks
-   Enable new workloads via Ray's superior flexibility.

We plan to implement all of the common patterns during prototype
validation, and will select some advanced patterns as we are closer to
release. Feedback on workload selection is welcome! The following
categories are in rough order of technical complexity:

### Common patterns: Microbenchmarks (simple distributed inference)

|                                                                 | Key properties / requirements                                        | Goals                                                            |
|-----------------------------------------------------------------|----------------------------------------------------------------------|------------------------------------------------------------------|
| vLLM tensor parallelism                                         |                                                                      | Reduce Ray overheads                                             |
| vLLM pipeline parallelism                                       | P2P/cross-node GPU communication                                     | Reduce (expected) Ray overheads; Validate cross-node performance  |



### Common patterns: Training workloads
|                                                                 | Key properties / requirements                                        | Goals                                                            |
|-----------------------------------------------------------------|----------------------------------------------------------------------|------------------------------------------------------------------|
| Tensor-parallel distributed training (TP)                       | Iterative                                                            | Performance parity                                               |
| Distributed data-parallel (DDP)                                 | Iterative; Collective ops that must be overlapped with backwards pass | Performance parity                                               |
| [GPipe](https://arxiv.org/abs/1811.06965) style pipeline-parallel distributed training (PP)         | Iterative P2P GPU communication                                      | Performance parity; Increase flexibility of partitioning scheme   |


### Advanced patterns
|                                                                 | Key properties / requirements                                        | Goals                                                            |
|-----------------------------------------------------------------|----------------------------------------------------------------------|------------------------------------------------------------------|
| [Pipedream](https://arxiv.org/pdf/1806.03377.pdf) style pipeline-parallel distributed training (PP)     | Iterative P2P GPU communication                                      | Performance parity; Increase flexibility of partitioning scheme   |
| vLLM pipeline parallelism on heterogeneous GPUs                 | Asymmetric compute                                                   | Reduce implementation burden                                     |
| Fault-tolerant distributed serving                              | Resume execution w/o restarting everyone                             | Reduce downtime via greater recovery flexibility                 |
| Fault-tolerant distributed training                             | Resume execution w/o restarting everyone and recover state           | Reduce downtime via greater recovery flexibility                 |
| Alpa                                                            | TP, DDP and PP training                                              | Reduce implementation burden                                     |
| Fully sharded data parallel (FSDP)                              | Overlap forwards pass with backwards pass                            | Performance parity; Increase flexibility of partitioning scheme   |
| Activation offloading to CPU                                    | Overlap GPU-CPU communication with other computation                 | Performance parity                                               |
| Speculative decoding for LLM inference                          | Model composition                                                    | Reduce Ray overheads; Increase flexibility of partitioning scheme |
| Disaggregated prefill for LLM inference                          | Dynamic scheduling | Reduce Ray overheads; Increase flexibility of scheduling |

### A note on fault tolerance

When ML workers fail, there may be a number of sources of overhead,
including:

-   Python/process initialization
-   Collective group initialization
-   Reloading model weights

Inference vs training also affects recovery time, e.g., reloading
weights for training might require everyone to roll back.

The proposed architecture is not a cure-all for these recovery
overheads. However, we do believe that it will improve flexibility and
reduce implementation burden. In particular:

-   Single-controller model allows the controller to coordinate
    operations such as failure detection and reconfiguration of layers
    across workers
-   Ability to reuse worker processes
-   Ability to pipeline various steps in recovery, by breaking recovery
    into smaller tasks that can be scheduled as a DAG