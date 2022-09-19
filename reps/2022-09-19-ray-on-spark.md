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
 
- Inside each spark executor, launch one Ray Node (head / worker) and keep the spark job running until
the Ray cluster destroyed. Note: one spark executor might contain multiple spark tasks,
but we will only launch one ray node for each spark executor.
e.g., suppose in executor1, there’re 3 spark tasks launched, then in task of local-rank-0,
we launch a ray node with –num-cpus=3. Using the following commands to launch ray node in spark tasks:
  - **Head node:** ray start —head --num-cpus=X --num-gpus=Y --memory=Z
  - **Worker node:** ray start —head --num-cpus=X --num-gpus=Y --memory=Z --address={head_ip}:{port}

 
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
Because we need to allocate compute resources to the ray head node, If launching in spark task side,
it might consume driver side resources, and if there are too many users sharing the spark cluster,
spark driver resources might be exhausted.
So, I suggest to launch Ray head node on spark task side, we can pick the first spark task to launch
head node.
Issue: How to send the ray head node IP / port to spark driver side ?
So that on spark driver side user can run  the ray application on the ray cluster.
Because the spark barrier job keeps running when the ray cluster is up, we cannot use
`rdd.collect()` to send the head node IP / port  back to driver side, we need to broadcast
the spark driver IP to task side and then in spark task, we can send the  head node IP / port
back to driver node by TCP connection.


#### How to select the port number used by Ray node ?
Ray node requires listening on several port, a spark cluster might be shared by many users,
each users might setup their own ray cluster concurrently, to avoid port conflicts,
we can randomly select free port and start Ray node service,
if failed we can retry on another free port.


#### How much memory shall we allocate to Ray node service ( set via ray script --memory option) ?
Spark does not provide explicit API for getting task allowed memory,
I propose:
SPARK_WORKER_MEMORY / SPARK_WORKER_CORES * RAY_NODE_NUM_CPUS
But how to get SPARK_WORKER_MEMORY and SPARK_WORKER_CORES in spark task is an issue.


#### How to make ray respect spark GPU resource scheduling ?
In spark task, we can get GPU IDs allocated to this task, so, when launching
Ray node, besides specifying `--num-gpus` options, we need specify `CUDA_VISIBLE_DEVICES`
environment so that we can restrict Ray node only uses the GPUs allocated to corresponding spark tasks.


### API Proposal

#### Initialize ray cluster on spark API

```
ray_cluster_address, ray_cluster_handler = ray.spark.init_cluster(num_cpus, num_gpus, memory)
```

Init a ray cluster on the spark cluster, the argumens specified the ray cluster can use how much cpus / gpus / memory.


Returns a `ray_cluster_address` string (Ray Head node IP / port) and a `ray_cluster_handler`


#### Initialize ray client API.

Now User can run ray application on the returned ray cluster, by executing `ray.init(address=ray_cluster_address)`, and then run any ray application code.


### Shutdown ray cluster on spark

When user want to shutdown the ray cluster, he can call:

```
ray_cluster_handler.shutdown()
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
