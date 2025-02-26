# Support RayService Incremental Upgrades

## General Motivation

Currently, during a zero-downtime upgrade on a RayService, the KubeRay operator temporarily creates a pending RayCluster and waits for it to become ready. Once ready, it switches all traffic to the new cluster and then terminates the old one. That is, both RayCluster custom resources require 100% computing resources. However, most users do not have access to such extensive resources, particularly in the case of LLM use cases where many users [lack sufficient high-end GPUs](https://github.com/ray-project/kuberay/issues/1476).

It'd be useful to support an API to incrementally upgrade the RayService, scaling a new RayCluster to handle only a % capacity of the total traffic to the RayService in order to avoid delays in the upgrade process due to resource constraints. By scaling the upgraded cluster incrementally, while the old cluster is scaled down by the same factor, Ray can enable a smoother traffic migration to the new service. In the case where a user deploys an upgraded RayService with a high load of requests, an incremental upgrade allows autoscaling to begin sooner on the new cluster, reducing the overall latency required for the service to handle the higher load.

In order to enable this feature, we propose the following APIs and integration with [Gateway](https://github.com/kubernetes-sigs/gateway-api).


### Should this change be within `ray` or outside?
Within Ray. For better code maintenance.

## Stewardship

### Shepherd of the Proposal
- @kevin85421

## Design and Architecture

### Proposed API

#### RayService spec

MaxSurgePercent (*int32 [0, 100])
- Definition: The maximum allowed percentage increase in capacity during a scaling operation. It limits how much the new total capacity can exceed the original target capacity. This field defines the "next state" for the upgraded RayCluster. Each iteration that `target_capacity` percent of total traffic is routed to the upgraded RayCluster, the upgraded RayCluster should increase its `target_capacity` by 100% + `MaxSurgePercent` - old RayCluster `target_capacity`.
- `MaxSurgePercent` must adhere to (old RayCluster `target_capacity` + new RayCluster `target_capacity`) <= (100% + `MaxSurgePercent`)
- Defaults to 100, which is the same as a blue/green deployment strategy

StepSizePercent (*int32 [0, 100])
- `StepSizePercent` represents the percentage of traffic to transfer to the upgraded cluster every `IntervalSeconds` seconds. `StepSizePercent` is therefore the increase in the route weight associated with the upgraded cluster endpoint. The percentage of traffic routed to the upgraded RayCluster will increase by `StepSizePercent` until equal to `target_capacity` of the upgraded RayCluster. At the same time, the percent of traffic routed to the old RayCluster will decrease by `StepSizePercent` every `IntervalSeconds` seconds.
- Required value if `IncrementalUpgrade` type is set

IntervalSeconds (*int32 [0, ...])
- `IntervalSeconds` represents the number of seconds for the controller to wait between increasing the percentage of traffic routed to the upgraded RayCluster by `StepSizePercent` percent.
- Required value if `IncrementalUpgrade` type is set

GatewayClassName (string)
- `GatewayClassName` represents the name of the Gateway Class installed in the cluster by the K8s cluster admin.

After adding the above fields to the Ray Serve schema, these APIs will be added to the KubeRay CR spec and can be specified by the user as follows:

A new `Type` called `IncrementalUpgrade` will be introduced for this change to specify an upgrade strategy with `MaxSurgePercent`, `StepSizePercent`, `Interval` set that enables an incremental upgrade:

```go
type RayServiceUpgradeType string

const (
	// During upgrade, NewCluster strategy will create new upgraded cluster and switch to it when it becomes ready
	NewCluster RayServiceUpgradeType = "NewCluster"
	// During upgrade, IncrementalUpgrade strategy will create an upgraded cluster to gradually scale
	// and migrate traffic to using Gateway API.
	IncrementalUpgrade RayServiceUpgradeType = "IncrementalUpgrade"
	// No new cluster will be created while the strategy is set to None
	None RayServiceUpgradeType = "None"
)
```

Additionally, the new API fields would be added to the `RayServiceUpgradeStrategy`:
```sh
type RayServiceUpgradeStrategy struct {
  // Type represents the strategy used when upgrading the RayService.
  // Currently supports `NewCluster`, `IncrementalUpgrade`, and `None`.
  Type *RayServiceUpgradeType `json:"type,omitempty"`
  // IncrementalUpgradeOptions defines the behavior of an IncrementalUpgrade.
  IncrementalUpgradeOptions *IncrementalUpgradeOptions `json:"type,omitempty"`
}
```

The behavior of the Incremental Upgrade can be configured through the following options:
```sh
type IncrementalUpgradeOptions struct {
  // The capacity of serve requests the upgraded cluster should scale to handle each interval.
  // Defaults to 100%.
  // +kubebuilder:default:=100
  MaxSurgePercent *int32 `json:"maxSurgePercent,omitempty"`
  // The percentage of traffic to switch to the upgraded RayCluster at a set interval after scaling by MaxSurgePercent.
  StepSizePercent *int32 `json:"stepSizePercent"`
  // The interval in seconds between transferring StepSize traffic from the old to new RayCluster.
  IntervalSeconds *int32 `json:"intervalSeconds"`
  // The name of the Gateway Class installed by the Kubernetes Cluster admin.
  GatewayClassName string `json:"gatewayClassName"`
}
```

Example RayService manifest:
```sh
apiVersion: ray.io/v1
kind: RayService
metadata:
  name: example-rayservice
spec:
  upgradeStrategy:
    type: "IncrementalUpgrade"
    incrementalUpgradeOptions:
      maxSurgePercent: 20  // the upgraded cluster will increase target_capacity by 20% each iteration
      stepSizePercent: 5  // represents 5% traffic to swap each interval seconds
      intervalSeconds: 10 // represents 10 second migration interval
      gatewayClassName: "cluster-gateway" // Created by K8s cluster admin
  serveConfigV2: |
    ...
```

#### RayService status

The Ray Serve schema currently has the field `target_capacity` with the following definition:

[target_capacity](https://github.com/ray-project/ray/blob/2ae9aa7e3b198ca3dbe5d65f8077e38d537dbe11/python/ray/serve/schema.py#L38) (int32 [0, 100]) - [Implemented in Ray 2.9](https://github.com/ray-project/ray/commit/86e0bc938989e28ada38faf25b75f717f5c81ed3)
- Definition: The percentage of traffic routed to the RayService the upgraded cluster is expected to accept after the upgrade.

We can also introduce a new field named `TrafficRoutedPercent` with the following definition:

TrafficRoutedPercent (int32 [0, 100])
- Definition: `TrafficRoutedPercent` is the percentage of traffic that the RayService is currently accepting. During an incremental upgrade, the percentage of traffic routed to the active RayService will be gradually migrated to the pending RayService.

The above fields can be exposed through the RayService status:
```sh
type RayServiceStatus struct {
    ...
    TargetCapacity *int32 `json:"targetCapacity,omitempty"`
    TrafficRoutedPercent *int32 `json:"trafficRoutedPercent,omitempty"`
}
```

The controller can then access the `TargetCapacity` and `TrafficRoutedPercent` for the old RayCluster and upgraded RayCluster through the `ActiveServiceStatus` and `PendingServiceStatus` respectively of the `RayServiceStatuses` in the RayService CR.


#### Key requirements:
- If users want to use the original blue/green deployment, RayService should work regardless of whether the K8s cluster has Gateway-related CRDs
- RayService incremental upgrade should be verified on a minimal Gateway controller implementation

### Gateway API

Kubernetes Gateway API is a set of resources designed to manage HTTP(S), TCP, and other network traffic for Kubernetes clusters. It provides a more flexible and extensible alternative to the traditional Ingress API, offering better support for service mesh integrations, routing policies, and multi-cluster configurations. Gateway API already has a well-defined interface for traffic splitting, so KubeRay would not need to implement backends for different Ingress controllers.

Gateway APIs:

| API |  Use   | Status |  K8s version  |
| ------- | ------ | ---------- | ------------- |
|  v1.GatewayClass | Defines a Gateway Cluster level resource | GA (v0.5+) | v1.24+ |
|  v1.Gateway | Infrastructure that binds Listeners to a set of IP addresses | GA (v0.5+) | v1.24+ |
|  v1.HTTPRoute | Provides a way to route HTTP requests | GA (v0.5+) | v1.24+ |
|  v1.GRPCRoute | Specifies routing behavior of gRPC requests from a Gateway listener to an API object | GA (v1.1+) | v1.25+ |


The responsibilities for managing the Gateway API resources associated with a RayService incremental upgrade are as follows:
- K8s cluster admin:
  - Gateway controller - supported controllers would be those with stable support for `HTTPRoute` and `GRPcRoute`
  - GatewayClass - defines the controller to use
  - Gateway CRDs

- RayService controller:
  - Gateway - created per RayService and defines the listeners and routes to use
  - HTTPRoute - defines how HTTP traffic is routed to the two RayCluster endpoints
  - GRPcRoute - defines how GRPc traffic is routed to the two RayCluster endpoints


### Example Upgrade Process

1. K8s cluster admin installs Gateway CRDs, Gateway Class, and Gateway controller that is compatible with their infrastructure backend.

2. A user creates a RayService with a `RayServiceUpgradeType` of `IncrementalUpgrade`.

3. The KubeRay controller validates the values of `maxSurgePercent`, `stepSizePercent`, and `intervalSeconds`.

4. KubeRay controller checks the Serve related fields (`containerPort`, etc.) of the RayService spec and creates a Gateway object.

Example Gateway:
```sh
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: <ray-service-name>-gateway
spec:
  gatewayClassName: gatewayClassName // passed in from RayService CR
  listeners:
  - name: http
    protocol: HTTP
    port: serve_container_port // set in RayService spec, defaults to 8000
```

5. The KubeRay controller requires the Ray autoscaler (resource level) to be enabled in order to support a gradual traffic migration that scales the upgraded cluster by `stepSizePercent` traffic routed through Gateway. We could validate that autoscaling is enabled in the previous step and accept/reject the RayService CR accordingly. Alternatively, the controller could flip the `enableInTreeAutoscaling` flag to enable node resource autoscaling and add some default values.

6. When a RayService with `IncrementalUpgrade` type is created, the KubeRay controller creates an `HTTPRoute` with settings according to the serve config. The `weight` associated with the upgraded cluster endpoint should start at 0.

Example `HTTPRoute`:
```sh
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httproute-<gateway-name>
spec:
  parentRefs:
  - kind: Gateway
    name: <gateway-name>
  hostnames:
  - "ray.example.com"
  rules:
  - matches:
    ...
    - backendRefs:
        - name: <head-svc-of-original-ray-cluster>
          weight: 1
          port: serve_container_port // defaults to 8000
```

Example `GRPcRoute` (we will start with HTTP first for alpha):
```sh
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpcroute-<gateway-name>
spec:
  parentRefs:
  - kind: Gateway
    name: <gateway-name>
  hostnames:
  - "ray.example.com"
  rules:
  - matches:
    ...
    - backendRefs:
        - name: <head-svc-of-original-ray-cluster>
          weight: 1
          port: serve_container_port // defaults to 9000
```

7. When an upgrade is initiated, KubeRay creates the upgraded RayCluster with the Ray autoscaler enabled and `target_capacity` set to 100% + `max_surge_percent` - old RayCluster `target_capacity`. The Ray autoscaler will scale up the appropriate number of worker group replicas until the Serve replicas on the upgraded cluster are scheduled and 'Ready'.

8. The controller should update the routes to prepare to switch traffic to the upgrading RayCluster using a `backendRef`:
```sh
backendRefs:
...
  - name: <head-svc-of-upgraded-ray-cluster>
    weight: 0
    port: serve_container_port
```

9. Once the upgraded RayCluster is ready (see related issue to allow users to define when Serve apps are "Ready": https://github.com/ray-project/ray/issues/50883), the KubeRay controller will increment the weight of the upgraded RayCluster `backendRef` by `stepSizePercent`, while decreasing the weight of the old RayCluster `backendRef` by `stepSizePercent` and then wait `intervalSeconds`. The controller should loop incrementing the traffic until the `weight` associated with the upgraded cluster is equal to its `target_capacity`. Gateway API will route the specified `weight` percentage of traffic to the old and new RayClusters accordingly.

10. Once the upgraded cluster is accepting the additonal `maxSurgePercent` capacity of requests (up to 100%), the controller can scale down the old RayCluster by decreasing `target_capacity` by `maxSurgePercent` and allowing the Ray autoscaler to reduce the size of the original cluster. In the last iteration, the new `target_capacity` should be increased to a maximum of 100% and the old RayCluster `target_capacity` should be decreased to 0%.

11. The controller will loop, increasing the `target_capacity` of the new RayCluster by `maxSurgePercent`, waiting for the Serve apps to become ready, and then performing steps 9 and 10. Once `target_capacity` of the new RayCluster is equal to 100%, the upgrade is complete and the KubeRay controller can delete the old RayCluster object and remove its `backendRef` from the routes. The Gateway API objects can be retained for future updates.


### Rollback Support

A key requirement for RayService incremental upgrade is to support rollback if the upgrade becomes stuck or leads to a bad state. During an upgrade, the KubeRay controller is aware of the old RayCluster state, upgraded RayCluster state, and the 'Goal State' or the state currently defined in the RayService CR. If users update the RayService CR during an incremental upgrade, the rollback process should be performed as follows:

1. The KubeRay controller checks whether the goal state has changed. If the state hasn't changed (i.e it still matches the upgraded cluster), continue the upgrade.

2. If the goal state has changed and matches the original RayCluster, perform a rollback by scaling the original RayCluster to 100% `target_capacity` and then switching 100% of traffic to the original cluster. The upgraded cluster can then be cleaned up.

3. If the goal state doesn't match either the original cluster or upgraded cluster, the controller should perform the rollback as defined in step #2 and then continue the upgrade as defined by the current goal state from the original cluster.


## Compatibility, Deprecation, and Migration Plan

### Compatibility

Below is a compatibility matrix for the proposed supported versions of Kubernetes, KubeRay, and Gateway API:

| KubeRay |  Ray   | Kubernetes |  Gateway API  |
| ------- | ------ | ---------- | ------------- |
|  v1.2.2 | 2.8.0+ |   1.23+    | Not Supported |
|  v1.3.0 | 2.9.0+ |   1.25+    | v1.1+ (v1)    |

Gateway API [guarantees support](https://gateway-api.sigs.k8s.io/concepts/versioning/#supported-versions) for a minimum of the 5 most recent Kubernetes minor versions (v1.27+). Gateway still supports Kubernetes v1.24+ but no longer tests against these older minor versions. The latest supported version is v1 as released by the [v1.2.1](https://github.com/kubernetes-sigs/gateway-api/releases/tag/v1.2.1) release of the Gateway API project. For KubeRay we should support a minimum of Kubernetes [v1.25+](https://kubernetes.io/blog/2023/10/31/gateway-api-ga/#cel-validation) to avoid the additional install of a CEL validating webhook.

The incremental upgrade proposal requires only standard Gateway APIs (`HTTPRoute`, `GRPcRoute`, and a `Gateway`) to be available from the KubeRay controller, and thus should be compatible with any Gateway controller that implements the standard APIs.

### Limitations and Assumptions

There are certain assumptions the KubeRay controller will make that users must follow or traffic being split has the possibility of being dropped. These include:
- If the K8s Cluster admin does not have sufficient node resources (i.e. at the cloud provider level) to schedule the new cluster on, it's possible for the upgrade to hang while it waits for `target_capacity` serve replicas to be scheduled on the upgraded RayCluster. Autoscaling settings such as `upscalingMode` and `idleTimeoutSeconds` which control how quickly the RayClusters are scaled, will enable the user to control the gradual migration of the incremental upgrade and minimize the possibility of increased latency during an upgrade.
- The KubeRay controller will assume that the presence of a Gateway controller and required CRDs are sufficient to enable an upgrade. The KubeRay controller will make no guarantees to support all Gateway controller instances (i.e. those in alpha status, etc.).

 
## Test Plan and Acceptance Criteria

We'll add unit and e2e tests in both Ray Core and KubeRay to test against the new API fields and Gateway CRDs. We should also add Kubernetes compatibility tests to ensure the compatibility matrix with Ray, KubeRay, and Gateway API is not broken.

The `IncrementalUpgrade` upgrade type will require a Gateway API compatible controller and CRDs to be installed prior to creating the RayService CR object. We plan to start with support for a minimal Gateway controller and add testing for additional controllers as needed. The minimal requirements would be to support weighted traffic splitting with `Gateway` and `HTTPRoute`, since these are the only APIs that will be used by the KubeRay controller.
