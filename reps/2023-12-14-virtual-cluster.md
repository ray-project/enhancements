# Virtual Cluster

## Summary

Ray currently has the [physical cluster](https://docs.ray.io/en/releases-2.9.0/cluster/getting-started.html) concept and we proposes to add a new virtual cluster concept. A virtual cluster is a partition of the physical cluster and can be dynamically scaled at runtime. A physical cluster can be partitioned into multiple virtual clusters and each virtual cluster runs a single Ray job. Through this way, multiple jobs can share the cluster resources with isolation. Virtual cluster is a fundamental building block for multi-tenant Ray.

<img src="https://user-images.githubusercontent.com/898023/291094699-35bac047-5844-4f2c-a794-17cd18e96219.png" alt="drawing" width="726"/>

### General Motivation

While users can run multiple Ray jobs within a single Ray cluster simultaneously, there is currently no sufficient isolation between jobs. Users also have no way to specify cross-job policies such as fairness. For example, if you run two Ray Tune jobs on the same cluster, they will both try to use all the cluster resources and compete for cluster resources in an ad-hoc mechanism, without notion of isolation, fairness, or priority. In order to properly support multi-tenancy, we need a machnism to share cluster resources between different jobs with isolation and certain cross-job policies. While [placement group](https://docs.ray.io/en/releases-2.9.0/ray-core/scheduling/placement-group.html) can solve some of the issues, its lack of nesting and autoscaling support makes it unusable for certain workloads. Virtual cluster is the mechanism we propose here and it supports isolation, nesting, and autoscaling.

### Should this change be within `ray` or outside?

Inside `ray` project since this is a Ray Core feature.

## Stewardship

### Required Reviewers

@ericl, @stephanie-wang, @scv119

### Shepherd of the Proposal

@ericl

## Design

With the introduction of virtual clusters, every Ray job runs in its own virtual cluster and only has access to resources inside that virtual cluster. Each virtual cluster has a spec that defines the min and max resources of the cluster. Min resources are minimal resources required for the job to run and they are atomically reserved for gang scheduling. If min resources cannot be reserved when there are not enough available resources, the job will be queued. With job queueing, we can implement different policies such as FIFO or priority-based queueing. Max resources are the autoscaling limit of the virtual cluster and the maximal resources can be used by the job.

Virtual clusters can be nested and a Ray job can create sub-clusters to isolate separate parts of its application workload. For example, a Tune grid sweep job can create a sub-cluster for each of its nested Train workload. These possibly nested virtual clusters form a tree where the root is the entire physical cluster.

<img src="https://user-images.githubusercontent.com/898023/291139618-0be11470-db09-466d-8c2c-37b9e5b3765c.png" alt="drawing" width="508"/>

Virtual clusters with different min and max resources are autoscalable. When scaling up, virtual clusters will try to borrow more resources from their parent virtual clusters. If their parents have available resources, the scaling up is instant. Otherwise, parents will try to borrow resources from their parents recursively and eventually this may cause the upscale of the physical cluster. When scaling down, virtual clusters will return resources to their parents recursively and eventually this may cause the downscale of the physical cluster.

A Ray physical cluster consists of a set of Ray physical nodes and, similarly, a virtual cluster consists of a set of virtual nodes. Each virtual node is a partition of a single physical node and it has resources and node labels just like the physical node. A virtual node can be either fixed size or flexible/resizable. For a single virtual cluster, there can be multiple fixed-size virtual nodes but at most one flexible virtual node on a single physical node.

### API

#### Virtual Cluster Spec

```
message VirtualCluster {
  // A virtual cluster consits of flexible resources and fixed size resources.

  // == Flexible resources ==
  // Defines flexible resource limit across the virtual cluster.
  // Ray will guarantee flexible resource usage does not exceed this limit.

  // If specified, ensure we have at least this min amount
  // of resources before starting the cluster.
  // If not specified, the default value is 0.
  map<string, double> flexible_resource_min

  // If specified, limit the consumption of these resources to
  // the specified values.
  // If not specified, the default value is infinite.
  map<string, double> flexible_resource_max

  // == Fixed size resources ==
  // Fixed sized resources to request, e.g. {"GPU": 4}.
  // These resources are part of the min resources
  // that will be atomically reserved when the
  // virtual cluster is created.
  repeated FixedSizeNodes fixed_size_nodes
}

message FixedSizeNode {
  map<string, double> resources
  
  // Additional labels that the
  // virtual node has in addition to
  // those inherited from the parent node.
  map<string, string> labels
}

enum SchedulingPolicy {
  PACK
  SPREAD
  STRICT_SPREAD
}

message FixedSizeNodes {
  repeated FixedSizeNode nodes
  // One of PACK, SPREAD, or STRICT_SPREAD. These would be
  // defined with respect to parent virtual nodes for nested
  // clusters.
  SchedulingPolicy scheduling_policy
}
```

#### Job API

Currently we have two ways to run a Ray job: ``ray.init()`` and ``ray job submit``. Both will take an optional parameter specifying the spec of the virtual cluster inside which the job will run. If unspecified, the default virtual cluster has zero min resources and infinite max resources meaning it can scale up to use the entire physical cluster resources.

```
# Default virtual cluster
# The job can use up to the entire physical cluster resources.
ray.init()

# The job can use at least 1 CPU and at most 8 CPUs.
ray.init(virtual_cluster=VirtualCluster(flexible_resource_min={"CPU": 1}, flexible_resource_max={"CPU": 8}))
```

```
# Default virtual cluster
# The job can use up to the entire physical cluster resources.
ray job submit -- python job.py

# The job needs 2 * 1 GPU that are strict spreaded.
ray job submit --virtual-cluster='{"fixed_size_nodes": [{"nodes": [{"resources": {"GPU": 1}}, {"resources": {"GPU": 1}}], "scheduling_policy": "STRICT_SPREAD"}]}' -- python job.py
```

Once a job is running inside a virtual cluster, it can use all the Ray APIs as if it's running inside its own Ray cluster.

#### Placement Group API

Since virtual clusters are nestable and support gang scheduling, they can be used to implement or replace placement groups.

```
# Create a placement group with two bundles that are packed
pg = placement_group([{"GPU": 4}, {"GPU": 1}], strategy="PACK")
# Run the task inside the placement group
task.options(scheduling_strategy=PlacementGroupSchedulingStrategy(placement_group=pg)).remote()

# Create a virtual cluster with two virtual nodes that are packed
vc = VirtualCluster(fixed_size_nodes=[FixedSizeNodes(nodes=[FixedSizeNode(resources={"GPU": 4}), FixedSizeNode(resources={"GPU": 1})], scheduling_policy=PACK)])
# A nested virtual cluster inside the job virtual cluster
with vc:
  # Run the task inside the virtual cluster
  task.remote()
```

```
# Create a placement group with two bundles that are strict spreaded
pg = placement_group([{"GPU": 4}, {"GPU": 1}], strategy="STRICT_SPREAD")
# Run the actor using the bundle 0 resources
Actor.options(scheduling_strategy=PlacementGroupSchedulingStrategy(placement_group=pg, placement_group_bundle_index=0)).remote()

# Create a virtual cluster with two virtual nodes that are strict spreaded
vc = VirtualCluster(fixed_size_nodes=[FixedSizeNodes(nodes=[FixedSizeNode(resources={"GPU": 4}, labels={"bundle_index": "0"}), FixedSizeNode(resources={"GPU": 1}, labels={"bundle_index": "1"})], scheduling_policy=STRICT_SPREAD)])
with vc:
  # Run the actor using the first virtual node resources
  Actor.options(node_labels={"bundle_index": In("0")}).remote()
```

Here we have two options for the API:

1. Deprecate the placement group API and use virtual cluster API directly.
2. Keep the placement group API and only change the internal implementation to use virtual cluster.

#### Examples

In this section, we show several Ray workloads and how they can be built on top of virtual clusters.
For each example, we show both the code using current Ray APIs and the code using the new virtual cluster APIs.

##### Gang scheduling a group of actors

```
ray.init()
pg = placement_group([{"GPU": 1}, {"GPU": 1}], strategy="STRICT_SPREAD")
actors = []
for i in range(2):
  actors.append(Actor.options(num_gpus=1, scheduling_strategy=PlacementGroupSchedulingStrategy(placement_group=pg, placement_group_bundle_index=i)).remote())
```

```
ray.init(virtual_cluster=VirtualCluster(fixed_size_nodes=[FixedSizeNodes(nodes=[FixedSizeNode(resources={"GPU": 1}, labels={"bundle_index": "0"}), FixedSizeNode(resources={"GPU": 1}, labels={"bundle_index": "1"})], scheduling_policy=STRICT_SPREAD)]))
actors = []
for i in range(2):
  actors.append(Actor.options(num_gpus=1, node_labels={"bundle_index": In(str(i))}).remote())
```

##### Tune + Dataset

```
# Tune can run 2 trails (each trail runs two 1 GPU trainers) in parallel and Dataset can use 1-10 CPUs
ray.init(virtual_cluster=VirtualCluster(fixed_size_nodes=[FixedSizeNodes(nodes=[FixedSizeNode(resources={"GPU": 1}), FixedSizeNode(resources={"GPU": 1})], scheduling_policy=PACK), FixedSizeNodes(nodes=[FixedSizeNode(resources={"GPU": 1}), FixedSizeNode(resources={"GPU": 1})], scheduling_policy=PACK)], flexible_resource_min={"CPU": 1}, flexible_resource_max={"CPU": 10}))
```

##### Multi-datasets

```
ray.init(virtual_cluster=VirtualCluster(flexible_resource_min={"CPU": 100}, flexible_resource_max={"CPU": 100}))

train_dataset_vc = VirtualCluster(flexible_resource_min={"CPU": 80}, flexible_resource_max={"CPU": 80})
with train_dataset_vc:
  ...

validation_dataset_vc = VirtualCluster(flexible_resource_min={"CPU": 20}, flexible_resource_max={"CPU": 20})
with validation_dataset_vc:
  ...
```

#### Cluster Introspection API

With virtual clusters, Ray jobs should have the illusion that they are running inside their own clusters exclusively. This means all the existing cluster instrospection APIs (e.g. ``ray.cluster_resources()``) need to return data that are only relevant to the current virtual cluster.

```
ray.cluster_resources():
  """This returns the total resources of the current virtual cluster."""

ray.available_resources():
  """This returns the available resources of the current virtual cluster."""

ray.nodes():
  """This returns all virtual nodes of the current virtual cluster."""

ray.util.state.summarize_tasks():
  """Summarize the tasks in the current virtual cluster."""

ray.util.state.summarize_objects():
  """Summarize the objects in the current virtual cluster."""

ray.util.state.*():
  """Only return state of the current virtual cluster."""
```

Besides the existing APIs, we also introduce more introspection APIs for virtual clusters.

```
class RuntimeContext:
  def current_cluster() -> VirtualClusterInfo:
    """Return the current virtual cluster this process is in.

    You can get the parent cluster using current_cluster().parent_cluster().
    """

  def current_node() -> VirtualNodeInfo:
    """Return the current virtual node this process is in."""

class VirtualClusterInfo:
    def spec() -> VirtualCluster
    def cluster_id() -> str
    def total_resources()  # like ray.cluster_resources()
    def max_resources()  # total after autoscaling to max, if known
    def available_resources()  # like ray.available_resources()
    def used_object_store_memory() # Object store memory used by objects in this virtual cluster
    def nodes()  # like ray.nodes()
    def state()  # like ray.util.state
    def child_clusters() -> List[VirtualClusterInfo]
    def parent_cluster() -> Optional[VirtualClusterInfo]

class VirtualNodeInfo:
    def node_id() -> str
    def node_labels() -> Dict[str, str]
    def used_resources() -> Dict[str, double]
    def avail_resources() -> Dict[str, double]
    def total_resources() -> Dict[str, double]
    def parent_node() -> Optional[VirtualNodeInfo]
```

### Implementation

#### Scheduling

Unlike the placement group implementation which is based on custom resources, virtual clusters and virtual nodes will be first-class citizen concept in Ray Core and we will use node labels to schedule tasks or actors to nodes.

With virtual clusters, each raylet will maintain a flatten list of local virtual nodes and resource view of remote virtual nodes.

```
class Raylet:
  local_nodes: List[Node]
  # Raylet id -> a list of virtual nodes on that raylet
  remote_nodes: Dict[str, List[Node]]

class Node:
  total_resources: Dict[str, float]
  available_resources: Dict[str, float]
  # Besides custom labels, each node will have two system labels
  # one is virtual node id label and the other is virtual cluster id label.
  # e.g. {"ray.io/vnode_id": "88888", "ray.io/vcluster_id": "66666"}
  labels: Dict[str, str]
```

When submitting a task or actor, Ray will automatically add the virtual cluster id label selector so that the task or actor can be scheduled to virtual nodes belonging to the current virtual cluster.

Let's walk through an example to understand this better. We first start with a Ray cluster of 2 nodes, each with 4 CPUs:

```
Raylet1:
  local_nodes: [Node(total_resources={"CPU": 4}, available_resources={"CPU": 4}, labels={"ray.io/vnode_id": "raylet1", "ray.io/vcluster_id": "physical_cluster_id"})]
  remote_nodes: {"raylet2": [Node(total_resources={"CPU": 4}, available_resources={"CPU": 4}, labels={"ray.io/vnode_id": "raylet2", "ray.io/vcluster_id": "physical_cluster_id"})]}

Raylet2:
  local_nodes: [Node(total_resources={"CPU": 4}, available_resources={"CPU": 4}, labels={"ray.io/vnode_id": "raylet2", "ray.io/vcluster_id": "physical_cluster_id"})]
  remote_nodes: {"raylet1": [Node(total_resources={"CPU": 4}, available_resources={"CPU": 4}, labels={"ray.io/vnode_id": "raylet1", "ray.io/vcluster_id": "physical_cluster_id"})]}
```

Now a Job with a fixed size virtual cluster (2 * 1 CPU, STRICT_SPREAD) is started:

```
Raylet1:
  local_nodes: [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet1", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 1}, labels={"ray.io/vnode_id": "vnode1", "ray.io/vcluster_id": "vcluster1"})]
  remote_nodes: {"raylet2": [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet2", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 1}, labels={"ray.io/vnode_id": "vnode2", "ray.io/vcluster_id": "vcluster1"})]}

Raylet2:
  local_nodes: [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet2", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 1}, labels={"ray.io/vnode_id": "vnode2", "ray.io/vcluster_id": "vcluster1"})]
  remote_nodes: {"raylet1": [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet1", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 1}, labels={"ray.io/vnode_id": "vnode1", "ray.io/vcluster_id": "vcluster1"})]}
```

Next we submit a 1 CPU task:

```
task.options(num_cpus=1).remote()

# This will be rewritten to
task.options(num_cpus=1, node_labels={"ray.io/vcluster_id": ray.get_runtime_context().current_cluster().cluster_id()}).remote()
```

When raylet1 (assuming it's the local raylet) receives the lease request, it will look at local nodes and remote nodes and find all nodes that match the node label selectors. In this case they are vnode1 and vnode2. Since vnode1 is local, it will choose vnode1 to run the task:

```
Raylet1:
  local_nodes: [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet1", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 0}, labels={"ray.io/vnode_id": "vnode1", "ray.io/vcluster_id": "vcluster1"})]
  remote_nodes: {"raylet2": [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet2", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 1}, labels={"ray.io/vnode_id": "vnode2", "ray.io/vcluster_id": "vcluster1"})]}

Raylet2:
  local_nodes: [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet2", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 1}, labels={"ray.io/vnode_id": "vnode2", "ray.io/vcluster_id": "vcluster1"})]
  remote_nodes: {"raylet1": [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet1", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 0}, labels={"ray.io/vnode_id": "vnode1", "ray.io/vcluster_id": "vcluster1"})]}
```

Next we submit another 1 CPU task. When raylet1 receives the lease request, there are still vnode1 and vnode2 that match the node label selectors but vnode1 has no available resources so raylet1 will spillback the lease request to raylet2 and raylet2 will choose vnode2 to run the task:

```
Raylet1:
  local_nodes: [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet1", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 0}, labels={"ray.io/vnode_id": "vnode1", "ray.io/vcluster_id": "vcluster1"})]
  remote_nodes: {"raylet2": [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet2", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 0}, labels={"ray.io/vnode_id": "vnode2", "ray.io/vcluster_id": "vcluster1"})]}

Raylet2:
  local_nodes: [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet2", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 0}, labels={"ray.io/vnode_id": "vnode2", "ray.io/vcluster_id": "vcluster1"})]
  remote_nodes: {"raylet1": [Node(total_resources={"CPU": 4}, available_resources={"CPU": 3}, labels={"ray.io/vnode_id": "raylet1", "ray.io/vcluster_id": "physical_cluster_id"}), Node(total_resources={"CPU": 1}, available_resources={"CPU": 0}, labels={"ray.io/vnode_id": "vnode1", "ray.io/vcluster_id": "vcluster1"})]}
```

If we submit another 1 CPU task, it will wait in the raylet task queue until one of the previous tasks finish.

Once the job finishes, the corresponding virtual cluster is destroyed and the resources are returned back to its parent.

```
Raylet1:
  local_nodes: [Node(total_resources={"CPU": 4}, available_resources={"CPU": 4}, labels={"ray.io/vnode_id": "raylet1", "ray.io/vcluster_id": "physical_cluster_id"})]
  remote_nodes: {"raylet2": [Node(total_resources={"CPU": 4}, available_resources={"CPU": 4}, labels={"ray.io/vnode_id": "raylet2", "ray.io/vcluster_id": "physical_cluster_id"})]}

Raylet2:
  local_nodes: [Node(total_resources={"CPU": 4}, available_resources={"CPU": 4}, labels={"ray.io/vnode_id": "raylet2", "ray.io/vcluster_id": "physical_cluster_id"})]
  remote_nodes: {"raylet1": [Node(total_resources={"CPU": 4}, available_resources={"CPU": 4}, labels={"ray.io/vnode_id": "raylet1", "ray.io/vcluster_id": "physical_cluster_id"})]}
```

