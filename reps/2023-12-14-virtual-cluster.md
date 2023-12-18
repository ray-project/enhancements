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

#### Cluster Introspection API

### Implementation
