## Summary
### General Motivation

[Ray on spark](https://github.com/ray-project/enhancements/blob/main/reps/2022-09-19-ray-on-spark.md) supports launching the Ray cluster over spark cluster. But currently, Ray on spark does not support Ray cluster autoscaling. It only supports starting a Ray cluster with fixed number of worker nodes.

The purpose of Ray on spark autoscaling is to support following scenarios:

- For a standalone mode apache spark cluster, assuming multiple users share the spark cluster, and one user submits a spark application that runs a Ray on spark cluster. If Ray on spark autoscaling is supported, and the spark application enables [dynamic resources allocation](https://spark.apache.org/docs/3.4.1/job-scheduling.html#dynamic-resource-allocation), then the Ray on spark cluster starts with zero Ray worker nodes. When Ray jobs are submitted, the Ray on spark cluster requests more Ray worker nodes based on demands (i.e. scales up). It will launch more spark executors for this spark application. After Ray jobs completes, then the Ray on spark cluster scales down. It will terminate idle Ray worker nodes and then trigger some idle spark executor termination. So that with Ray on spark autoscaling feature enabled, when the Ray on spark cluster is idle, it does not occupy spark worker resources.

- For a databricks spark cluster that enables [databricks cluster autoscaling](https://docs.databricks.com/en/clusters/cluster-config-best-practices.html#autoscaling), with Ray on spark autoscaling enabled, we can start the databricks spark cluster with zero initial spark workers. When Ray jobs are submitted, it triggers Ray on spark cluster scaling up and the underlying databricks spark cluster scaling up. When Ray jobs complete, it triggers Ray on spark cluster scaling down and the underlying databricks spark cluster scaling down. This feature helps databricks users to improve cloud resource utilization and save cost.


#### Key requirements:
- Supports Apache Spark cluster that is configured with [standalone mode](https://spark.apache.org/docs/3.4.1/spark-standalone.html).
- Support Apache Spark application that enables dynamic resource allocation.
- Supports databricks spark cluster that enables spark worker autoscaling.


### Should this change be within `ray` or outside?
Within Ray. For better code maintenance.


## Stewardship
### Required Reviewers

- @jjyao
- @ericl

### Shepherd of the Proposal (should be a senior committer)

- @jjyao

## Design and Architecture

### Prerequisites

- The user has an active Spark cluster (apache/spark >= 3.3).

- The user must install Ray packages on every node of the Spark cluster, e.g. via:

  ```
  pip install ray
  ```

  If the user's workload requires Ray add-ons, they must also be installed, e.g. via

  ```
  pip install "ray[debug,dashboard,tune,rllib,serve]"
  ```

### Ray on spark autoscaling interfaces

Integrate autoscaling feature into existing `ray.util.spark.cluster_init.setup_ray_cluster` API,
The following new arguments are added:

 - autoscale: bool (default False)
   If True, enable autoscaling, the number of initial Ray worker nodes 
   is zero, and the maximum number of Ray worker nodes is set to
   `num_worker_nodes`. Default value is False.
 - autoscale_upscaling_speed: Optional[float] (default 1.0)
   The upscaling speed configuration, see explanation in [here](https://docs.ray.io/en/latest/cluster/vms/user-guides/configuring-autoscaling.html#upscaling-and-downscaling-speed).
 - autoscale_idle_timeout_minutes: Optional[float] (default 1.0)
   The downscaling speed configuration, see explanation in [here](https://docs.ray.io/en/latest/cluster/vms/user-guides/configuring-autoscaling.html#upscaling-and-downscaling-speed).
 

### Implementation

To start a Ray cluster that enables autoscaling, firstly, prepares a config file like following:

```yaml
cluster_name: spark
max_workers: 8  # maximum number of workers
provider:
    # This is the key of registered Ray autoscaling `NodeProvider` class that is
    # used as the backend of launching Ray worker nodes.
    type: spark
    # This must be true since the nodes share the same ip!
    use_node_id_as_ip: True
    disable_node_updaters: True
    disable_launch_config_check: True
available_node_types:
    ray.head.default:  # Ray head configurations
        resources:
            CPU: 4
            GPU: 0
        node_config: {}
        max_workers: 0
    ray.worker:  # Ray worker configurations
        resources:
            CPU: 4
            GPU: 2
            memory: 2000000000
            object_store_memory: 1000000000
        node_config: {}
        min_workers: 0  # minimum number of workers in this group
        max_workers: 4  # maximum number of workers in this group
head_node_type: ray.head.default
upscaling_speed: 1.0
idle_timeout_minutes: 0.1
```

Then, starts a Ray head node via following command (some options are omitted):

```shell
ray start --head --autoscaling-config=/path/to/autoscaler-config.yaml
```

The above command starts a Ray head node, it also starts a Ray autoscaler with
above YAML configuration, it configures autoscaler with "spark" `NodeProvider`
backend. When Ray cluster is requested to scale up,
the configured `NodeProvider` is called to trigger Ray worker node setup.

This is `NodeProvider` [interfaces](https://github.com/ray-project/ray/blob/e1c48ab8b1225daa44ea63ed1e7dc956b0e7f411/python/ray/autoscaler/node_provider.py#L13).

### How to implement spark node provider?

For each Ray worker node, we start a spark job with only one spark
task to hold this Ray worker node, and we set a unique spark job group ID to
this spark job. When we need to terminate this Ray worker node, we cancel
the corresponding spark job by canceling corresponding spark job group, so that
the spark job and its spark task are killed, then it triggers the Ray worker node termination.

Because we can only cancel spark job not spark task when we need to scale down a Ray worker node.
So we have to have one spark job for each Ray worker node.

One thing critical is that spark node provider runs in autoscaler process that is different process
than the one that executes "setup_ray_cluster" API. User calls "setup_ray_cluster" in
spark application driver node, and the semantic is "setup_ray_cluster" requests spark resources from
this spark application. Internally, "setup_ray_cluster" should use "spark session" instance
to request spark application resources. But spark node provider runs in another python process,
how to share current process spark session to the separate NodeProvider process? The solution is
setting up a spark job server that runs inside spark application driver process (the process
that calls "setup_ray_cluster" API), and in NodeProvider process, it sends RPC request to
the spark job server for creating spark jobs in the spark application. Note that we cannot
create another spark session in NodeProvider process, because if doing so, it means we create
another spark application, and then it causes NodeProvider requests resources belonging to
the new spark application, but we need to ensure all requested spark resources are belong to
the original spark application that calls "setup_ray_cluster" API.

The overall architecture is:

![ray on spark autoscaling architecture](https://github.com/ray-project/ray/assets/19235986/9f1f30bb-e395-4d98-8c8d-5fa60675bafa)


### Key questions


#### We can use `ray up` command to set up a ray cluster with autoscaling, why we don't call `ray up` command in ray on spark autoscaling implementation ?

In ray on spark, we only provides a python API `setup_ray_cluster` and it does not have a CLI. So in `setup_ray_cluster` implementation, we need to generate autoscale config YAML file according to `setup_ray_cluster` argument values, and then launch the ray head node with "--autoscaling-config" option. In this way ray on spark code can manage ray head node process and ray worker nodes easier.

The second reason is ray-on-spark autoscaling only supports a very limited subset of cluster.yaml: e.g. only a single worker type, no docker, no ssh so it's easier and less confusing to use a restricted API.

#### How to integrate Ray on spark NodeProvider backend with databricks spark cluster autoscaling feature ?

See [Cluster size and autoscaling](https://docs.databricks.com/en/clusters/configure.html#autoscaling) and [Enhanced Autoscaling](https://docs.databricks.com/en/delta-live-tables/auto-scaling.html) for configuring databricks spark cluster autoscaling.

Note that scaling up or down is triggered automatically. In Ray on spark NodeProvider,
we just need to create or cancel spark jobs on demand, then databricks cluster
will automatically scale up or down according to cluster resources utilization ratio.


#### How to integrate Ray on spark NodeProvider backend with apache/spark application dynamic resource allocation on standalone mode spark cluster ?

See [dynamic resources allocation](https://spark.apache.org/docs/latest/job-scheduling.html#dynamic-resource-allocation) for configuring spark application dynamic resource allocation.

Note that scaling up or down is triggered automatically. In Ray on spark NodeProvider
backend, we just need to create or cancel spark jobs by demand, then the spark application executors will automatically scale up or down according to this spark application resources utilization ratio.


#### How to make `NodeProvider` backend support multiple Ray worker nodes running on the same virtual machine ?

By default, `NodeProvider` implementation implement `internal_ip` and `external_ip` methods and convert `node_id` to IP, and different node must have different IP address,
and Ray autoscaler internal code uses IP to track specific node status.

But, for Ray on spark, one virtual machine (running a spark worker) might start multiple
ray worker nodes. It breaks rules above and the workaround is that we set `use_node_id_as_ip` configuration to `True` in autoscaler configuration file:

```yaml
provider:
    type: spark
    use_node_id_as_ip: True
    ...
```

and we use incremental number as the node id used in NodeProvider implementation code. Before creating Ray worker nodes, we set a special resource `NODE_ID_AS_RESOURCE` and the total quantity is the value of the node id. Ray autoscaler code can then track these Ray nodes status correctly.
See related logic code [here](https://github.com/ray-project/ray/blob/7a8b6a1b52488922fc27a1bc2a01d40f87d36af6/python/ray/autoscaler/_private/monitor.py#L304C34-L304C34) for reference.


#### When `NodeProvider.create_node_with_resources_and_labels` is triggered, and node creation request is emitted, but it takes several minutes to launch a new spark worker node and create ray worker node, during the creation pending period, how to notify Ray autoscaler that the Ray worker node is pending creation, but not creation failure ?

Once `NodeProvider.create_node_with_resources_and_labels` is called, node creation request is emitted, and in spark node provider, this node status is marked as "setting-up". Once the ray worker node is actually launched, spark node provider changes the node status to "up-to-date". If it detects the ray worker node is down, spark node provider removes the node out of active node list.


#### Ray autoscaler supports setting multiple Ray worker groups, each Ray worker group has its individual CPU / GPU / memory resources configuration, and its own minumum / maximum worker number setting for autoscaling. Shall we support this feature in Ray on spark autoscaling?

Current use-cases only require all Ray worker nodes having the same shape,
we can support this in future if customer requires it.


#### What's the default Ray on spark minimum worker number we should use ?

I propose to set it to zero. Ray worker node launching is very quick, and setting it to
zero increases cluster resources utilization ratio especially when the Ray on spark cluster is often idle.


#### Apache/spark dynamic resource allocation also supports Spark over Kubernetes/Yarn, can we make Ray on spark autoscaling supports Spark over Kubernetes/Yarn ?

It is doable, but currently Ray on spark only supports apache/spark standalone mode. Once Ray on spark supports Spark over Kubernetes/Yarn, we can make Ray on spark autoscaling feature support it.
