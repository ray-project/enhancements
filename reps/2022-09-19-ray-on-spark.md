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

TODO

### Shepherd of the Proposal (should be a senior committer)

TODO

## Design and Architecture

### Preparation

We need to have a spark cluster (apache/spark >= 3.3)
We need install ray packages on every node of the cluster, by:
```
pip install ray
pip install "ray[debug,dashboard,tune,rllib,serve]"
```

### How to setup Ray cluster over spark cluster ?

A spark cluster is like:
![spark-cluster-overview](https://spark.apache.org/docs/latest/img/cluster-overview.png)

Setup ray cluster over spark cluster, we can do:
- Create a spark barrier mode job,  a spark barrier mode job means all tasks in this spark job 
will be executed concurrently, and each task is allocated with several resources, by default,
one task is allocate with 1 cpu core. If we want to setup a Ray cluster with  16 cpu cores resources
in total, we can create a spark barrier mode with 16 tasks.
 
- In spark driver side, launch Ray head node, by:
```
ray start —head --num-cpus=0
```
This makes the Ray head node only works as a manager node, without worker node functionality.

- Inside each spark executor, launch one Ray worker worker and keep the spark job running until
the Ray cluster destroyed. Each ray worker node will be assigned with resources equal to the corresponding
spark task (CPU / GPU / Memory). The command used to start ray node is:
```
ray start —head --num-cpus=X --num-gpus=Y --memory=Z --address={head_ip}:{port}
```

- After Ray cluster launched, the Ray application can be submitted to the ray cluster via
the Ray Head Node address / port.
 
- When user want to shutdown the Ray cluster, cancel the spark job and kill all ray node services. 

By this approach, in a spark cluster, we can create multiple isolated Ray clusters, each cluster
uses resources restricted to each spark job allocated resources.


### Key questions

#### We use spark barrier mode job to launch a ray cluster, shall we make the spark barrier to be a background job and keep running, until user explicitly terminate it ?

Yes. So every user only need to start ray cluster once and then user can run ray applications
on the ray cluster with little overhead.


#### Launch Ray head node on spark driver node or spark task side ?
In spark driver side.
Because we can set Ray head node with `num-cpus` = 0, so that no ray task will be scheduled
to ray head node, so that we can launch ray head node in driver side, it won't consume
too much computing / memory resources of driver node.


#### Shall we make ray worker node the same shape (assigned with the same resources amount) ?
Yes. Otherwise, the Ray cluster setup will be nondeterministic,
and you could get very strange results with bad luck on the node sizing.


#### how to deterministically config resources per node ?
In each spark task, launch one ray worker node. Each spark task has the same assigned resources shape.


#### What's recommended ray node shape ?

cpu cores >= 4 and memory >= 10 GB
This corresponding to:
spark.task.cpus config >= 4 and (SPARK_WORKER_MEMORY / TASK_SLOTS_PER_SPARK_WORKER) >= 10GB


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


#### What’s the level of isolation spark provides ?
Resources isolation.
Each ray cluster are isolated and will use its own resources allocated to the corresponding spark job.


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


For each ray node, it might have different / random CPUs assigned ,
but for the whole ray cluster, the resources amount allocated is deterministic.

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
