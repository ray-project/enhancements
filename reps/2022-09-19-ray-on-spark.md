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

We intend to represent each Ray worker node as a long-running Spark
task in which Ray worker nodes have the same set of resources as the Spark task initiating each of 
the Ray workers. Multiple tasks can be run on a single Spark worker node, meaning that 
multiple Ray worker nodes can run on a single Spark worker node. 
Finally, we intend to run the Ray head node on the Spark driver node with a fixed resource allocation.

Note that if multiple ray-node-per-spark-worker issues occur (whether due to shared object store 
location for workers, dashboard visualization confusion, or other unforeseen issues), the implementation 
can be modified to only permit a single Ray cluster per Spark cluster. 

To clarify further, the following example demonstrates how a Ray cluster with 16 total worker CPU
cores can be launched on a Spark cluster, assuming that each Ray node requires 4 CPU cores and
10GB of memory:

1. On the Spark driver node, launch the Ray head node as follows:

```
ray start --head --num-cpus=0
```

This ensures that CPU processing tasks will not execute on the head node.

2. Create a Spark job that executes all tasks in the spark job in standard map-reduce mode. Each
task is allocated a fixed number of CPUs and a fixed amount of memory. In this example, we will
allocate 4 CPU cores to each Spark task and at least 10GB of memory (by ensuring that each Spark
worker node has at least 10GB of memory reserved for every set of 4 CPU cores, as computed by
``SPARK_WORKER_MEMORY`` / ``TASK_SLOTS_PER_WORKER`` under the assumption that the
``spark.task.cpus`` configuration is set to ``4``).

3. In each constituent Spark task, launch one Ray worker node and allocate to it
the full set of resources available to the Spark task (4 CPUs, 10GB of memory). Keep the Spark task
running until the Ray cluster is destroyed. The command used to start each Ray worker node is as
follows:

```
ray start --num-cpus=X --num-gpus=Y --memory=Z --object-store-memory=M --address={head_ip}:{port}
```

4. After the Ray cluster is launched, the user's Ray application(s) can be submitted to the Ray
cluster via the Ray head node address / port.

5. To shut down the Ray cluster, cancel the Spark job and terminate all Ray node
services that are running on the Spark driver and Spark workers.

Finally, by adjusting the resource allocation for Ray nodes and Spark tasks, this approach enables
users to run multiple Ray clusters in isolation of each other on a single Spark cluster, if desired.


### Key questions


#### Launch Ray head node on spark driver node or spark task side ?

The head node will be initialized on the Spark Driver. To ensure that tasks will not be submitted 
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
Ray node, besides specifying `--num-gpus` options, we need to specify `CUDA_VISIBLE_DEVICES`
environment so that we can restrict Ray node only uses the GPUs allocated to corresponding spark tasks.


#### How to support all options of ray start
Provide a ray_node_options argument (dict type).


#### Where should the code live: ray repo or spark repo?
Ray repo.
Putting in spark repo is hard, It would be very hard to pass vote.
Past examples includes horovod, petastorm, xgboost, tensorflow-spark, all of them tried to put in spark repo but failed.


#### Does it support autoscaling ?
We will be investigating and testing the feasibility of supporting autoscaling of ray worker nodes as 
additional spark task slots become available upon the initiation of additional spark worker nodes.

#### What is the fault tolerance of Barrier Execution mode and how are failures handled?
The current design uses standard job scheduling in Spark and will initiate task submission to new 
Spark worker nodes in the event that a task group fails, similar to the functionality within Spark 
during a task retry.

#### What’s the level of isolation spark provides ?
Spark task execution provides process-level isolation.
No VM/container level isolation. 


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
