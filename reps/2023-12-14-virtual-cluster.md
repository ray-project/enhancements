# Virtual Cluster

## Summary

Ray currently has the [physical cluster](https://docs.ray.io/en/releases-2.9.0/cluster/getting-started.html) concept and we proposes to add a new virtual cluster concept. A virtual cluster is a partition of the physical cluster and can be dynamically scaled at runtime. A physical cluster can be partitioned into multiple virtual clusters and each virtual cluster runs a single Ray job. Through this way, multiple jobs can share the cluster resources with isolation. Virtual cluster is a fundamental building block for multi-tenancy Ray.

<img src="https://user-images.githubusercontent.com/898023/291094699-35bac047-5844-4f2c-a794-17cd18e96219.png" alt="drawing" width="726"/>

### General Motivation

While users can run multiple Ray jobs within a single Ray cluster simultaneously, there is currently no sufficient isolation between jobs. Users also have no way to specify cross-job policies such as fairness. For example, if you run two Ray Tune jobs on the same cluster, they will both try to use all the cluster resources and compete for cluster resources in an ad-hoc mechanism, without notion of isolation, fairness, or priority. In order to properly support multi-tenancy, we need a machnism to share cluster resources between different jobs with isolation and certain cross-job policies. While [placement group](https://docs.ray.io/en/releases-2.9.0/ray-core/scheduling/placement-group.html) can solve some of the issues, its lack of nesting and autoscaling support makes it unusable for certain workloads. Virtual cluster is the mechanism we propose here and it supports isolation, nesting and autoscaling.

### Should this change be within `ray` or outside?

Inside `ray` project since this is a Ray Core feature.

## Stewardship

### Required Reviewers
The proposal will be open to the public, but please suggest a few experienced Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@ericl, @stephanie-wang, @scv119

### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

@ericl

## Design

### API

#### Cluster Introspection API

### Implementation
