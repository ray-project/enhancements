## Summary
### General Motivation

[Ray on spark](https://github.com/ray-project/enhancements/blob/main/reps/2022-09-19-ray-on-spark.md) supports launch the Ray cluster over spark cluster, and the Ray cluster resource requests are converted to spark cluster resources requests. But currently, Ray on spark does not support Ray cluster autoscaling, it only supports starting a Ray cluster with fixed number of worker nodes, and all worker nodes are configured with a fixed shape, i.e., all Ray on spark worker nodes are configured with the same CPU / GPU / memory resources config.

The purpose of Ray on spark autoscaling is to support following scenarios:

- For a spark cluster with the fixed number spark workers, assuming multiple users share the spark cluster, without Ray on spark autoscaling, if one user starts a Ray on spark cluster with fixed number Ray worker nodes, but he does not run any Ray jobs on this Ray on spark cluster, then the spark cluster resources allocated to the Ray on spark cluster are locked and cannot be allocated to other spark cluster jobs until the Ray on spark cluster terminates. But, if Ray on spark cluster supports autoscaling, we can make the Ray on spark cluster starts with zero Ray worker nodes, and when Ray jobs are submitted, then the Ray on spark cluster requests more Ray worker nodes by demands (i.e. scales up), after Ray jobs completes, then the Ray on spark cluster scales down. So that with autoscaling enabled, when the Ray on spark cluster is idle, it does not occupy spark worker resources.

- For a databricks spark cluster that enables spark worker autoscaling, with Ray on spark autoscaling enabled, we can start the databricks spark cluster with zero initial spark workers, and when Ray jobs are submitted, it triggers Ray on spark cluster scaling up, then it triggers databricks spark cluster scales up. Vice versa, when Ray jobs complete, it triggers Ray on spark cluster scaling down, then it triggers databricks spark cluster scales down. This feature helps databricks users to improve cloud resource utilization.


#### Key requirements:
- Supports apache/spark cluster that is configured with [standalone mode](https://spark.apache.org/docs/latest/spark-standalone.html)
- Supports databricks spark cluster that enables spark worker autoscaling


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

- The user have an active Spark cluster (apache/spark >= 3.3)

- The user must install Ray packages on every node of the Spark cluster, e.g. via:

  ```
  pip install ray
  ```

  If the user's workload requires Ray add-ons, they must also be installed, e.g. via

  ```
  pip install "ray[debug,dashboard,tune,rllib,serve]"
  ```

### Ray autoscaler plugin interfaces

To start a Ray cluster that enables autoscaling, firstly, prepares a config file like following:

```yaml
cluster_name: ray_on_spark
max_workers: 8  # maximum number of workers
provider:
    # This is the key of registered Ray autoscaling `NodeProvider` class that is
    # used as the backend of launching Ray worker nodes.
    type: ray_on_spark
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
above YAML configuration, it configures autoscaler with "ray_on_spark" `NodeProvider`
backend, when Ray jobs are submitted Ray cluster is requested to scale up,
the configured `NodeProvider` backend is called to trigger Ray worker node setup.

The `NodeProvider` [interfaces](https://github.com/ray-project/ray/blob/e1c48ab8b1225daa44ea63ed1e7dc956b0e7f411/python/ray/autoscaler/node_provider.py#L13) are defined like following (only some important interfaces are listed here):

```python

class NodeProvider:

    def create_node_with_resources_and_labels(
        self, node_config, tags, count, resources, labels
    ):
        """
        Create `count` number of nodes, resources are node CPU / GPU / memory
        requirements that are defined in autoscaler config YAML file.
        After node creation, generate a unique node_id and link the node id with
        several node metadata like tags / labels.
        """
        ...
    
    def terminate_node(self, node_id):
        """
        Terminate node by provided node id.
        """
  
    def external_ip(self, node_id):
        """
        Get node external ip. In Ray on spark NodeProvider implementation,
        we need to use fake IP that equals to node id, see Key questions
        "How to make `NodeProvider` backend support multiple Ray worker nodes
        running on the same virtual machine"
        """
        ...

    def internal_ip(self, node_id):
        """
        Get node internal ip. In Ray on spark NodeProvider implementation,
        we need to use fake IP that equals to node id, see Key questions
        "How to make `NodeProvider` backend support multiple Ray worker nodes
        running on the same virtual machine"
        """
        ...


    def is_running(self, node_id):
        """
        query node running status.
        """
        ...

    def is_terminated(self, node_id):
        """
        query whether node is terminated
        """
        ...

    def node_tags(self, node_id):
        """
        query node tags
        """
        ...

    def set_node_tags(self, node_id, tags):
        """
        set node tags
        """
        ...

    @staticmethod
    def bootstrap_config(cluster_config):
        """
        Override original autoscaler configs here.
        """
        ...
```

### How to implement Ray on spark node provider backend ?

For Ray worker node creation, each Ray worker node, we start a spark job with only one spark
task to hold this Ray worker node. When we need to terminate this Ray worker node, we cancel
the corresponding spark task so that the Ray worker node is terminated.

The critical issue is, in the process that executing `NodeProvider` backends, how to get
the spark session in parent process ? We can set up a spark job server that runs inside
parent process with mulitple threaded mode. The overall architecture is:

![ray on spark autoscaling architecture](https://github.com/ray-project/ray/assets/19235986/9f1f30bb-e395-4d98-8c8d-5fa60675bafa)


### Key questions

#### How to make `NodeProvider` backend support multiple Ray worker nodes running on the same virtual machine ?

one node multiple worker nodes. node ip ?


Use one spark job or one task per job ?

how to start spark job ? spark job server ?

How to handle node creation time gap ?

spark task retries ?



how to pass arguments to autoscaler ?

shall we support different shape worker nodes ?

minimal Ray cluster worker number.
