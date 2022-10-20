## Summary
### General Motivation

Supporting running Ray applications on spark cluster / databricks runtime,
Spark is a popular distributed computing framework and is used widely,
If supporting running Ray applications on spark cluster, user don't need to
setup a standalone ray cluster and it allows ray workloads and spark workloads
runs together.

#### Key requirements:
- Ray application resources (cpu/gpu/memory) allocation must respect spark job resource allocation.
- Little overhead compared with native Ray cluster.
- (Optional) Ray UI portal support.

### Should this change be within `ray` or outside?

Yes. For better code maintemance.

## Stewardship
### Required Reviewers

- @jjyao
- @ericl

### Shepherd of the Proposal (should be a senior committer)

- @jjyao

## Design and Architecture

### Prerequisites

- The user have an active Spark cluster (apache/spark >= 3.3)

- The user must install Ray packages on every node of the Spark cluster, e.g. via:

  ```
  pip install ray
  ```

  If the user's workload requires Ray add-ons, they must also be installed, e.g. via

  ```
  pip install "ray[debug,dashboard,tune,rllib,serve]"
  ```

### How to setup Ray cluster over spark cluster ?

The architecture of a Spark cluster is as follows:
![spark-cluster-overview](https://spark.apache.org/docs/latest/img/cluster-overview.png)

Using Spark's barrier mode, we intend to represent each Ray worker node as a long-running Spark
task, where the Ray worker node has the same set of resources as the Spark task. Multiple Spark
tasks can run on a single Spark worker node, which means that multiple Ray worker nodes can run
on a single Spark worker node. Finally, we intend to run the Ray head node on the Spark driver
node with a fixed resource allocation.

To clarify further, the following example demonstrates how a Ray cluster with 16 total worker CPU
cores can be launched on a Spark cluster, assuming that each Ray node requires 4 CPU cores and
10GB of memory:

1. On the Spark driver node, launch the Ray head node as follows:

```
ray start --head --num-cpus=0
```

The Ray head node will operate solely as a  manager node, without worker node functionality.

2. Create a Spark barrier mode job, which executes all tasks in the spark job concurrently. Each
task is allocated a fixed number of CPUs and a fixed amount of memory. In this example, we will
allocate 4 CPU cores to each Spark task and at least 10GB of memory (by ensuring that each Spark
worker node has at least 10GB of memory reserved for every set of 4 CPU cores, as computed by
``SPARK_WORKER_MEMORY`` / ``TASK_SLOTS_PER_WORKER`` under the assumption that the
``spark.task.cpus`` configuration is set to ``4``).

3. In each constituent Spark task, launch one Ray worker node in blocking mode and allocate to it
the full set of resources available to the Spark task (4 CPUs, 10GB of memory). Keep the Spark task
running until the Ray cluster destroyed. The command used to start each Ray worker node is as
follows:

```
ray start —head --num-cpus=X --num-gpus=Y --memory=Z --address={head_ip}:{port}
```

4. After the Ray cluster is launched, the user's Ray application(s) can be submitted to the Ray
cluster via the Ray head node address / port.

5. To shut down the Ray cluster, cancel the Spark barrier mode job and terminate all Ray node
services that are running on the Spark driver and Spark workers.

Finally, by adjusting the resource allocation for Ray nodes and Spark tasks, this approach enables
users to run multiple Ray clusters in isolation of each other on a single Spark cluster, if desired.

Due to the fact that execution is performed via Spark BarrierExecution, a failed task (e.g., a 
node loss event, a heartbeat timeout in the Spark cluster manager service for an executor, etc.) the 
Spark task executing the ray job will be retried as a single concurrent unit. Because of this mode of 
operation and the inability to perform map-reduce task resumption for this initial design, recovery 
by a resumption of tasks 'in-place' is not possible.

### Key questions

#### We use spark barrier mode job to launch a ray cluster, shall we make the spark barrier to be a background job and keep running, until user explicitly terminate it ?

Yes. So every user only need to start ray cluster once and then user can run ray applications
on the ray cluster with little overhead.


#### Launch Ray head node on spark driver node or spark task side ?

The head node will be initialized on the Spark Driver. To ensure that tasks will not submitted 
to this node, we will configure `num-cpus`=0 for the head node. 
Additionally, in order to ensure that the head node is running on hardware sufficient to ensure 
that requisite ray system processes, a validation of minimum hardware configuration requirements will 
be performed prior to initialization of the ray cluster (minimum CPU cores, memory requirements)


#### Shall we make ray worker node the same shape (assigned with the same resources amount) ?
Yes. Otherwise, the Ray cluster setup will be nondeterministic,
and you could get very strange results with bad luck on the node sizing.


#### Do Ray nodes have a 1:1 mapping with Spark tasks or with Spark workers?
In each Spark task, we will launch one Ray worker node. All spark tasks are allocated the same
number of resources. As a result, all Ray worker nodes will have homogeneous resources because Spark
task resource configurations must be homogeneous.

#### What is the recommended minimal resource allocation of a Ray node?

Each Ray node should have at least 4 CPU cores and 10GB of available memory. This corresponds to
a Spark task configuration of:

- ``spark.task.cpus >= 4`` and
- ``(SPARK_WORKER_MEMORY / TASK_SLOTS_PER_SPARK_WORKER) >= 10GB``


#### On a shared spark cluster, shall we let each user launch individual ray cluster, or one user can create a shared ray cluster and other user can also use it ?
I think we can provide 2 API to support both cases.
Shared mode:
```ray.spark.get_or_create_shared_cluster```
and
```ray.spark.init_private_cluster```

And if one shared mode ray cluster already exists, new ray cluster is not allowed to be launched.


#### How to select the port number used by Ray node ?
Ray node requires listening on several port, a spark cluster might be shared by many users,
each users might setup their own ray cluster concurrently, to avoid port conflicts,
we can randomly select free port and start Ray node service,
if failed we can retry on another free port.


#### How much memory shall we allocate to Ray node service ( set via ray script --memory option) ?
Spark does not provide explicit API for getting task allowed memory,
SPARK_WORKER_MEMORY / SPARK_WORKER_CORES * RAY_NODE_NUM_CPUS


#### How will we allocate memory for the Ray object store?
We will calculate the memory reserved for the Ray object store in each Ray node as follows:

1. Calculate the available space in the ``/dev/shm`` mount.
2. Divide the ``/dev/shm`` available space by the number of Spark tasks (Ray nodes) that can run
   concurrently on a single Spark worker node, obtaining an upper bound on the per-Ray-node object
   store memory allocation.
3. To provide a buffer and guard against accidental overutilization of mount space, multiply the
   upper bound from (2) by ``0.8``.

This allocation will ensure that we do not exhaust ``/dev/shm`` storage and accidentally spill to
disk, which would hurt performance substantially.


#### How to make ray respect spark GPU resource scheduling ?
In spark task, we can get GPU IDs allocated to this task, so, when launching
Ray node, besides specifying `--num-gpus` options, we need specify `CUDA_VISIBLE_DEVICES`
environment so that we can restrict Ray node only uses the GPUs allocated to corresponding spark tasks.


#### How to support all options of ray start
Provide a ray_node_options argument (dict type).


#### Where should the code live: ray repo or spark repo?
Ray repo.
Putting in spark repo is hard, It would be very hard to pass vote.
Past examples includes horovod, petastorm, xgboost, tensorflow-spark, all of them tried to put in spark repo but failed.


#### Does it support autoscaling ?
Currently no. This implementation relies on spark barrier mode job, which does not support autoscaling currently.
In a future iteration, we plan to investigate (if warranted) using the Spark job scheduler to handle the 
creation of ray worker nodes by mapping a single ray worker to a Spark job running in shared resource context (non-FIFO).

#### What is the fault tolerance of Barrier Execution mode and how are failures handled?
BarrierRDD mapping forces all configured tasks to execute concurrently. With no concept of map-reduce 
fault tolerance in this mode, faults that occur within processing (that are not related to loss of 
an underlying VM that the Spark worker node and, by extension, the Ray worker node are running on) will 
be retried as an atomic unit. That is, all pending task transactions that are submitted to the ray cluster 
on all nodes will re-execute. 
We intend to development a retry loop within the BarrierRDD mapping function to allow for a limited 
number of retries of a task submission before raising an Exception up the stack to the Spark driver node 
to terminate the ray cluster. 
As mentioned above, a loss of a Spark worker is unrecoverable and will require a new initialization of a 
ray cluster once the dropped node(s) are recovered by new VMs. 
Note that this hardware-loss lack of fault tolerance will be mitigated if we end up pursuing a NodeProvider approach 
in the future (wherein a new Spark Job would be submitted to a recovered VM worker node to resume execution in a retry loop).

#### What’s the level of isolation spark provides ?
Barrier mode spark task provides process-level isolation.
No VM/container level isolation. Spark task does not support it.


#### Custom resources support
TPUs, specific accelerators, etc.
Uncommon scenario, We can do it later?. Spark doesn't support this, spark can only schedule CPU, GPU, memory.


### API Proposal

#### Initialize ray cluster on spark API

```
ray_cluster_on_spark = ray.spark.init_cluster(num_cpus, num_gpus, memory)
```

Or

```
ray_cluster_on_spark = ray.spark.init_cluster(num_spark_tasks)
```

Initialize a ray cluster on the spark cluster, the arguments specified the ray cluster can use how much cpus / gpus / memory.
And connect to the cluster.

Returns an instance of type `RayClusterOnSpark``

The ray head node may be of a different configuration compared to that of the worker nodes, but 
the ray workers will be homogenous with respect to CPU / GPU and memory available for heap and 
object store utilization.
A best-effort mapping of available Spark cluster resources to requested ray cluster resources for 
worker nodes will be performed. A validation check during initialization will be done to:
- In `safe_mode'=True raise an Exception if the configured resources are insufficient, providing detailed reconciliation steps for the user.
- In `safe_mode`=False, log a warning of potentially insufficient resources with instructions on how to configure the Spark cluster to avoid this situation.


e.g., your case: (8-CPU nodes, or 4-GPU nodes),
suppose on a spark cluster with config:

spark.task.cpus 2 # means 2 cpu cores per spark task
spark.task.resource.gpu.amount 1 # means 1 GPU per spark task
Then the Ray on spark routine will create a spark job with 4 spark tasks running concurrently,
which means it books (8-CPU nodes, or 4-GPU nodes) resources for ray cluster.


### Shutdown ray cluster on spark

When user want to shutdown the ray cluster, he can call:

```
ray_cluster_on_spark.shutdown()
```

It will terminate the ray cluster.
On databricks notebook, we can make databricks runtime automatically calls `ray_cluster_handler.shutdown()` when a notebook is detached from spark cluster. We need to install a hook to achieve this.


## Compatibility, Deprecation, and Migration Plan

Support apache/spark >= 3.3 and latest ray version.


## Test Plan and Acceptance Criteria

### Test Plan

- Setup ray cluster on spark cluster of configs and then run ray applications
  - 1 cpu per spark task
  - 1 cpu and 1 gpu per spark task
  - multiple cpu / gups per spark task

- Concurrently setup multiple ray cluster on a spark cluster

### Acceptance Criteria

- Ray application comply with spark cluster resource (cpu / gpu / memory) allocation.
- Less than 30% performance overhead.
