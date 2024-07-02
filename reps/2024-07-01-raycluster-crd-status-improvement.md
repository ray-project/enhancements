# RayCluster Status improvement

Author: @kevin85421, @rueian, @andrewsykim

## Context

CRD status is useful for observability. There are several GitHub issues and Slack threads about this topic. For example,
* https://github.com/ray-project/kuberay/issues/2182
* https://github.com/ray-project/kuberay/issues/1631
* https://github.com/ray-project/kuberay/pull/1930
* https://ray-distributed.slack.com/archives/C02GFQ82JPM/p1718207636330739
* https://ray-distributed.slack.com/archives/C01DLHZHRBJ/p1718261392154959
* @smit-kiri: “We've run into a issues where `RayJob` is stuck in `Initializing` state when the head / worker pods aren't able to spin up - like CrashLoopBackOff / ImagePullBackOff / not being able to pull a secret / etc”

Currently, the RayCluster state machine is quite messy, and the [states](https://github.com/ray-project/kuberay/blob/8453f9b6c749abf3b72b09e0d41032bbfcf17367/ray-operator/apis/ray/v1/raycluster_types.go#L112-L116) are not well-defined.

For example, when all Pods in the RayCluster are running for the first time, the RayCluster CR’s `Status.State` will transition to `ready` ([code](https://github.com/ray-project/kuberay/blob/8453f9b6c749abf3b72b09e0d41032bbfcf17367/ray-operator/apis/ray/v1/raycluster_types.go#L113)). However, the state will remain in ready even if we delete the head Pod. KubeRay will then create a new head Pod and restart the worker Pods.

Take another `ClusterState rayv1.Failed` ([code](https://github.com/ray-project/kuberay/blob/8453f9b6c749abf3b72b09e0d41032bbfcf17367/ray-operator/apis/ray/v1/raycluster_types.go#L114)) as an example, the RayCluster CR transitions to `rayv1.Failed` in many different scenarios, including when the YAML spec is invalid, when it fails to connect to the Kubernetes API server, when it fails to list information from the local informer, and so on. However, it doesn’t make sense to mark a RayCluster as `rayv1.Failed` when the KubeRay operator fails to communicate with the Kubernetes API server or fails to list local informer.

## Proposal

By design, the `RayCluster CRD` is similar to a bunch of Kubernetes `ReplicaSets`; both the head group and worker groups are analogous to `ReplicaSets`. Therefore, we can refer to [appsv1.ReplicaSetStatus](https://pkg.go.dev/k8s.io/api/apps/v1#ReplicaSetStatus).

This doc wants to propose several changes:

1. Remove `rayv1.Failed` from `Status.State`.
    * As mentioned in the 'Context' section, currently, there are many conditions for KubeRay to update a RayCluster’s state to `Failed`, including when KubeRay fails to connect to the Kubernetes API server and fails to list information from the local informer, which are not related to the RayCluster.
    * The [ReplicaSetStatus](https://pkg.go.dev/k8s.io/api/apps/v1#ReplicaSetStatus) doesn’t have the concept of `Failed`. Instead, it uses the `[]ReplicaSetCondition`. Reference:
        * [ReplicaSetConditionType](https://github.com/kubernetes/api/blob/857a946a225f212b64d42c68a7da0dc44837636f/apps/v1/types.go#L915)
        * [DeploymentConditionType](https://github.com/kubernetes/api/blob/857a946a225f212b64d42c68a7da0dc44837636f/apps/v1/types.go#L532-L542)
        * [Kueue](https://github.com/kubernetes-sigs/kueue)
    * It’s pretty hard to define `Failed` for RayCluster CRD.
        * For example, if you are using Ray Core to build your Ray applications, a worker Pod failure may cause your Ray jobs to fail. In this case, we may consider the RayCluster as `Failed` when a worker Pod crashes.
        * If users use Ray Train or Ray Data, these Ray AI libraries provide some application-level fault tolerance. To elaborate, if some Ray tasks/actors on the worker Pods fail, these libraries will launch new Ray tasks/actors to recover the job. In this case, we may consider the RayCluster as `Failed` when the head Pod crashes.
        * If users use Ray Serve with RayService and enable GCS fault tolerance for high availability, the RayCluster should still be able to serve requests even when the head Pod is down. Should we still consider the RayCluster as `Failed` when the head Pod crashes?

2. Introduce `Status.Conditions` to surface errors with underlying Ray Pods. 
    * TODO: We still need to define which types of information we want to surface in the CR status, operator logs, or Kubernetes events.

3. Make suspending a RayCluster an atomic operation.
    * Like RayJob, RayCluster also supports the [suspend](https://github.com/ray-project/kuberay/pull/1711) operation. However, the suspend operation is [atomic](https://github.com/ray-project/kuberay/pull/1798) in RayJob. In RayCluster, if we set suspend to true and then set it back to false before the RayCluster transitions to the state [rayv1.Suspended](https://github.com/ray-project/kuberay/blob/8453f9b6c749abf3b72b09e0d41032bbfcf17367/ray-operator/apis/ray/v1/raycluster_types.go#L115), there might be some side effects.

4. Add a new condition `Status.Conditions[HeadReady]` to indicate whether the Ray head Pod is ready for requests or not. This is also useful for RayJob and RayService, where KubeRay needs to communicate with the Ray head Pod.

5. Redefine `rayv1.Ready`.
    * As mentioned in the “Context” section, when all Pods in the RayCluster are running for the first time, the RayCluster CR’s `Status.State` will transition to `ready` ([code](https://github.com/ray-project/kuberay/blob/8453f9b6c749abf3b72b09e0d41032bbfcf17367/ray-operator/apis/ray/v1/raycluster_types.go#L113)). It is highly possible that the RayCluster will remain in the `ready` state even if some incidents occur, such as manually deleting the head Pod.
    * Here, we redefine `ready` as follows:
        * When all Pods in the RayCluster are running for the first time, the RayCluster CR’s `Status.State` will transition to `ready`.
        * After the RayCluster CR transitions to `ready`, KubeRay only checks whether the Ray head Pod is ready to determine if the RayCluster is `ready`.
        * This new definition is mainly to maintain backward compatibility.
    * TODO
        * I am considering making `ready` a condition in `Status.Conditions` and removing `rayv1.Ready` from `Status.State`. Kubernetes Pod also makes a [PodCondition](https://pkg.go.dev/k8s.io/api/core/v1#PodConditionType), but this may be a bit aggressive.
        * In the future, we can expose a way for users to self-define the definition of `ready` ([#1631](https://github.com/ray-project/kuberay/issues/1631)).

## Implementation

### Step 1: Introduce Status.Conditions

```go
type RayClusterStatus struct {
       ...
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

type RayClusterConditionType string

const (
	HeadReady RayClusterConditionType = "HeadReady"
	// Pod failures. See DeploymentReplicaFailure and
// ReplicaSetReplicaFailure for more details.
RayClusterReplicaFailure RayClusterConditionType = "ReplicaFailure"
)
```

* Future works
    * Add `RayClusterAvailable` or `RayClusterReady` to `RayClusterConditionType` in the future for the RayCluster readiness.
        * Reference: [DeploymentAvailable](https://github.com/kubernetes/api/blob/857a946a225f212b64d42c68a7da0dc44837636f/apps/v1/types.go#L532-L542), [LeaderWorkerSetAvailable](https://github.com/kubernetes-sigs/lws/blob/557dfd8b14b8f94633309f6d7633a4929dcc10c3/api/leaderworkerset/v1/leaderworkerset_types.go#L272)
    * Add `RayClusterK8sFailure` to surface the Kubernetes resource failures that are not Pods.

### Step 2: Remove `rayv1.Failed` from `Status.State`.
* Add the information about Pod failures to `RayClusterReplicaFailure` instead.

### Step 3: Make sure every reconciliation which has status change goes through `inconsistentRayClusterStatus`, and we only call `r.Status().Update(...)` when `inconsistentRayClusterStatus` returns true.

```go
if r.inconsistentRayClusterStatus(oldStatus, newStatus) {
// This should be the one and only place to call Status().Update(...)
// in RayCluster controller. 
	if err := r.Status().Update(...); err != nil {
		...
	}
}
```

* Currently, the status update in the RayCluster controller is quite random. For example, if some errors are thrown, the function `rayClusterReconcile` returns immediately and updates some parts of the RayCluster CR status randomly. Sometimes it doesn’t update anything; sometimes it updates `Status.State` and/or `Status.Reason`.
* Currently, the RayCluster controller calls `r.Status().Update(...)` in 3 places without any reason. Ideally, we should only call it in a single place for the whole controller. See RayJob controller as an example.

### Step 4: Add `PodName` and `ServiceName` to `HeadInfo`. 

```go
type HeadInfo struct {
	...
	PodName string `json:"podName,omitempty"`
	ServiceName string `json:"serviceName,omitempty"`
}
```

* `ServiceName` has already been added by [#2089](https://github.com/ray-project/kuberay/pull/2089).
* If we want to retrieve head Pod or head service in RayJob and RayService controllers, we should use `RayCluster.Status.Head.{PodName|ServiceName}` instead of something like [this function](https://github.com/ray-project/kuberay/blob/a43217bd2864961ee188a134807df57fd06e0a77/ray-operator/controllers/ray/rayservice_controller.go#L1223-L1235). 

### Step 5: Use `Condition[HeadReady]` instead of [isHeadPodRunningAndReady](https://github.com/ray-project/kuberay/blob/a43217bd2864961ee188a134807df57fd06e0a77/ray-operator/controllers/ray/rayservice_controller.go#L1211-L1217).

### Step 6: Redefine `rayv1.Ready`
* As mentioned in the “Context” section, when all Pods in the RayCluster are running for the first time, the RayCluster CR’s `Status.State` will transition to `ready` ([code](https://github.com/ray-project/kuberay/blob/8453f9b6c749abf3b72b09e0d41032bbfcf17367/ray-operator/apis/ray/v1/raycluster_types.go#L113)). It is highly possible that the RayCluster will remain in the `ready` state even if some incidents occur, such as manually deleting the head Pod.
* Here, we redefine `ready` as follows:
    * When all Pods in the RayCluster are running for the first time, the RayCluster CR’s `Status.State` will transition to `ready`.
    * After the RayCluster CR transitions to `ready`, KubeRay only checks whether the Ray head Pod is ready to determine if the RayCluster is `ready`.
    * This new definition is mainly to maintain backward compatibility.

### Step 7: Remove `rayv1.Suspended` from `Status.State`. Make suspending a RayCluster an atomic operation.

```go
type RayClusterStatus struct {
       ...
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

type RayClusterConditionType string

const (
    RayClusterSuspending RayClusterConditionType = "Suspending"
    RayClusterSuspended RayClusterConditionType = "Suspended"
)
```

* Introduce `RayClusterSuspending` and `RayClusterSuspended` to `Status.Conditions`. Then, we can refer to [RayJob](https://github.com/ray-project/kuberay/pull/1798) to make the suspend operation atomic.
