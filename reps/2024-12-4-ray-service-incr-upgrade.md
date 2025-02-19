# Support RayService Incremental Upgrades

## General Motivation

Currently, during a zero-downtime upgrade on a RayService, the KubeRay operator temporarily creates a pending RayCluster and waits for it to become ready. Once ready, it switches all traffic to the new cluster and then terminates the old one. That is, both RayCluster custom resources require 100% computing resources. However, most users do not have access to such extensive resources, particularly in the case of LLM use cases where many users [lack sufficient high-end GPUs](https://github.com/ray-project/kuberay/issues/1476).

It'd be useful to support an API to incrementally upgrade the RayService, scaling a new RayCluster to handle only a % capacity of the total traffic to the RayService in order to avoid delays in the upgrade process due to resource constraints. By scaling the upgraded cluster incrementally, while the old cluster is scaled down by the same factor, Ray can enable a smoother traffic migration to the new service. In the case where a user deploys an upgraded RayService with a high load of requests, an incremental upgrade allows autoscaling on the new cluster to begin sooner on the new cluster, reducing the overall latency required for the service to handle the higher load.

In order to enable this, we propose the following APIs and integration with [Gateway](https://github.com/kubernetes-sigs/gateway-api).


### Should this change be within `ray` or outside?
Within Ray. For better code maintenance.

## Stewardship

### Shepherd of the Proposal
- @kevin85421

## Design and Architecture

### Proposed API

[target_capacity](https://github.com/ray-project/ray/blob/2ae9aa7e3b198ca3dbe5d65f8077e38d537dbe11/python/ray/serve/schema.py#L38) (int32 [0, 100]) - [Implemented in Ray 2.9](https://github.com/ray-project/ray/commit/86e0bc938989e28ada38faf25b75f717f5c81ed3)
- Definition: (num_current_serve_replicas / num_expected_serve_replicas) * 100%
- Defaults to 100
- Ray Serve / RayService assumes that both the old and new RayClusters have the same capacity to handle requests. Consequently, the new RayCluster can handle up to target_capacity_new of the total requests.

rate (int32 [0, 100])
- `rate` represents the percentage of traffic to transfer to the upgraded cluster every `interval` seconds. `rate` is therefore the increase in the `HTTPRoute` and `GRPcRoute` weight associated with the upgraded cluster endpoint. The percentage of traffic routed to the upgraded RayCluster will increase by `rate` until `target_capacity` is reached. At the same time, the percent of traffic routed to the old RayCluster will decrease by `rate` every `interval` seconds.

interval (int32 [0, ...])
- `interval` represents the number of seconds for the controller to wait between increasing the percentage of traffic routed to the upgraded RayCluster by `rate` percent.

After adding the above field to the Ray Serve schema, these APIs will be added to the KubeRay CR spec and can be specified by the user as follows:

A new `type` called `IncrementalCluster` will be introduced for this change to specify an upgrade strategy with `TargetCapacity` or `MaxSurgePercent` set that enables an incremental upgrade:

```
type RayServiceUpgradeType string

const (
	// During upgrade, NewCluster strategy will create new upgraded cluster and switch to it when it becomes ready
	NewCluster RayServiceUpgradeType = "NewCluster"
	// During upgrade, IncrementalCluster strategy will create an additional, upgraded cluster with `target_capacity`
	// and route requests to both the old cluster and new cluster with Gateway.
	IncrementalCluster RayServiceUpgradeType = "IncrementalCluster"
	// No new cluster will be created while the strategy is set to None
	None RayServiceUpgradeType = "None"
)
```

Additionally, the new API fields would be added to the `RayServiceUpgradeStrategy`:
```sh
type RayServiceUpgradeStrategy struct {
  // Type represents the strategy used when upgrading the RayService.
  // Currently supports `NewCluster`, `IncrementalCluster`, and `None`.
  Type *RayServiceUpgradeType `json:"type,omitempty"`
  // The target capacity percentage of the upgraded RayCluster.
  // Defaults to 100% target capacity.
  // +kubebuilder:default:=100
  TargetCapacity *int32 `json:"targetCapacity,omitempty"`
  // The rate of traffic to route to the upgraded RayCluster.
  Rate *int32 `json:"rate"`
  // The interval in seconds between transferring Rate traffic from the old to new RayCluster.
  Interval *int32 `json:"interval"`
}
```

The RayService spec would then look as follows:
```sh
// RayServiceSpec defines the desired state of RayService
type RayServiceSpec struct {
  // UpgradeStrategy defines the scaling policy used when upgrading the RayService.
  UpgradeStrategy *RayServiceUpgradeStrategy `json:"upgradeStrategy,omitempty"`
  // Defines the applications and deployments to deploy, should be a YAML multi-line scalar string.
  ServeConfigV2   string         `json:"serveConfigV2,omitempty"`
  RayClusterSpec  RayClusterSpec `json:"rayClusterConfig,omitempty"`
  ...
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
    type: "IncrementalCluster"
    rate: 5      // represents 5%
    interval: 10 // represents 10 seconds
  serveConfigV2: |
    ...
```

#### Key requirements:
- If users want to use the original blue/green deployment, RayService should work regardless of whether the K8s cluster has Gateway-related CRDs
- RayService incremental upgrade should be verified on a minimal Gateway controller implementation
- If users specify`target_capacity`, they should be able to increase the value incrementally to scale up the new cluster until the upgrade is complete and the old RayCluster can be deleted

### Gateway API

Kubernetes Gateway API is a set of resources designed to manage HTTP(S), TCP, and other network traffic for Kubernetes clusters. It provides a more flexible and extensible alternative to the traditional Ingress API, offering better support for service mesh integrations, routing policies, and multi-cluster configurations. Gateway API already has a well-defined interface for traffic splitting, so KubeRay would not need to implement backends for different Ingress controllers.

Gateway APIs:

| API |  Use   | Status |  K8s version  |
| ------- | ------ | ---------- | ------------- |
|  v1.GatewayClass | Defines a Gateway Cluster level resource | GA (v0.5+) | v1.24+ |
|  v1.Gateway | Infrastructure that binds Listeners to a set of IP addresses | GA (v0.5+) | v1.24+ |
|  v1.HTTPRoute | Provides a way to route HTTP requests | GA (v0.5+) | v1.24+ |
|  v1.GRPCRoute | Specifies routing behavior of gRPC requests from a Gateway listener to an API object | GA (v1.1+) | v1.25+ |

### Example Upgrade Process

1. K8s cluster admin installs Gateway CRDs and Gateway controller that is compatible with their infrastructure backend.

2. A user creates a RayService with a `RayServiceUpgradeType` of `IncrementalCluster`.

3. The KubeRay controller validates the values of `target_capacity`, `rate`, and `interval` and then checks whether the Gateway CRDs and controller exists using the Kubernetes python client. If the controller doesn't exist, defaults to a blue/green deployment.

4. KubeRay controller checks the [Serve config files](https://docs.ray.io/en/releases-2.7.0/serve/production-guide/config.html#serve-config-files-serve-build) and creates a Gateway. Specifically, the controller validates the `http_options.port` and `grpc_options.port` to use for the respective listeners. If `grpc_servicer_functions` are missing, the GRPc listener can be ommitted.

Example Gateway:
```sh
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: <ray-service-name>-gateway
spec:
  gatewayClassName: <ray-service-name>-gateway
  listeners:
  - name: http
    protocol: HTTP
    port: http_options.port // defaults to 8000
 - name: grpc // can be ommitted if grpc_servicer_functions is empty
    protocol: GRPc
    port: grpc_options.port // defaults to 9000
```

5. The KubeRay controller requires both the Ray Serve autoscaler (application level) and Ray autoscaler (resource level) to be enabled in order to support a gradual traffic migration that scales the upgraded cluster by `rate` to serve `target_capacity` traffic routed through Gateway. We could validate that autoscaling is enabled in the previous step and accept/reject the RayService CR accordingly. Alternatively, at this step the controller could check the `autoscaling_config` of the Serve config and add some default values to enable incremental upgrade if missing. The controller could flip the `enableInTreeAutoscaling` flag to enable node resource autoscaling.

6. KubeRay creates the upgraded RayCluster according to the `autoscalerOptions` of the `rayClusterConfig` (i.e. initial replicas = `minReplicas`). Once the new RayCluster is ready, the KubeRay controller creates `HTTPRoute`s and `GRPcRoute`s with settings according to the serve config. The `weight` associated with the upgraded cluster endpoint should start at 0.

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
    - path:
        type: PathPrefix
        value: <route_prefix> // set according to serve config
        // add more paths according to `applications` in serve config
    - backendRefs:
        - name: <head-svc-of-original-ray-cluster>
          weight: 1
          port: http_options.port // defaults to 8000
       - name: <head-svc-of-upgraded-ray-cluster>
          weight: 0
          port: http_options.port // defaults to 8000
```

Example `GRPcRoute`:
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
    - path:
        type: PathPrefix
        value: <route_prefix> // set according to serve config
        // add more paths according to `applications` in serve config
    - backendRefs:
        - name: <head-svc-of-original-ray-cluster>
          weight: 1
          port: grpc_options.port // defaults to 9000
       - name: <head-svc-of-upgraded-ray-cluster>
          weight: 0
          port: grpc_options.port // defaults to 9000
```

7. Once the routes are created, the KubeRay controller increments the weight of the upgraded RayCluster `backendRef` by `rate`, while decreasing the weight of the old RayCluster `backendRef` by `rate`, waiting `interval` seconds between each iteration. Gateway API will route the specified `weight` percentage of traffic to the old and new RayClusters accordingly, allowing the Ray autoscaler to make scaling decisions for worker replicas and node resources in each RayCluster accordingly.

8. Once the `weight` of the traffic routed to the upgraded RayCluster is equal to `target_capacity`, the KubeRay controller stops increasing traffic at each interval. If `target_capacity` is equal to 100, the KubeRay controller can remove the created routes and delete the old RayCluster object. The Gateway object can be retained for future updates.

## Compatibility, Deprecation, and Migration Plan

### Compatibility

Below is a compatibility matrix for the proposed supported versions of Kubernetes, KubeRay, and Gateway API:

| KubeRay |  Ray   | Kubernetes |  Gateway API  |
| ------- | ------ | ---------- | ------------- |
|  v1.2.2 | 2.8.0+ |   1.23+    | Not Supported |
|  v1.3.0 | 2.9.0+ |   1.25+    | v1.1+ (v1)    |

Gateway API [guarantees support](https://gateway-api.sigs.k8s.io/concepts/versioning/#supported-versions) for a minimum of the 5 most recent Kubernetes minor versions (v1.27+). Gateway still supports Kubernetes v1.24+ but no longer tests against these older minor versions. The latest supported version is v1 as released by the [v1.2.1](https://github.com/kubernetes-sigs/gateway-api/releases/tag/v1.2.1) release of the Gateway API project. For KubeRay we should support a minimum of Kubernetes [v1.25+](https://kubernetes.io/blog/2023/10/31/gateway-api-ga/#cel-validation) to avoid the additional install of a CEL validating webhook.

### Limitations and Assumptions

There are certain assumptions the KubeRay controller will make that users must follow or traffic being split has the possibility of being dropped. These include:
- Since the upgraded RayCluster will be scaled based on routed traffic, if the K8s Cluster admin does not have sufficient node resources (i.e. at the cloud provider level) to schedule the new cluster on, it's possible for Pods to be left in a pending state and traffic to be eventually dropped. The `rate` that the new cluster scales up, the `interval` to wait between traffiic migrations, and autoscaling settings such as `idleTimeoutSeconds` which control how quickly the old cluster is scaled down, will all enable the user to control the gradual migration of the incremental ugprade and minimize the possibility of dropped traffic or increased latency.
- The KubeRay controller will assume that the presence of a Gateway controller and required CRDs are sufficient to enable an upgrade. The KubeRay controller will make no guarantees to support all Gateway controller instances (i.e. those in alpha status, etc.).

 
## Test Plan and Acceptance Criteria

We'll add unit and e2e tests in both Ray Core and KubeRay to test against the new API fields and Gateway CRDs. We should also add Kubernetes compatibility tests to ensure the compatibility matrix with Ray, KubeRay, and Gateway API is not broken.

The `IncrementalCluster` upgrade type will require a Gateway API compatible controller and CRDs to be installed prior to creating the RayService CR object. We plan to start with support for a minimal Gateway controller and add testing for additional controllers as needed. The minimal requirements would be to support weighted traffic splitting with `Gateway`, `HTTPRoute`, and `GRPcRoute`, since these are the only APIs that will be used by the KubeRay controller.
