# Support RayService Incremental Upgrades

## General Motivation

Currently, during a zero-downtime upgrade on a RayService, the KubeRay operator temporarily creates a pending RayCluster and waits for it to become ready. Once ready, it switches all traffic to the new cluster and then terminates the old one. That is, both RayCluster custom resources require 100% computing resources. However, most users do not have access to such extensive resources, particularly in the case of LLM use cases where many users [lack sufficient high-end GPUs](https://github.com/ray-project/kuberay/issues/1476).

It'd be useful to support an API to incrementally upgrade the RayService, scaling a new RayCluster to handle only a % capacity of the total traffic to the RayService in order to avoid delays in the upgrade process due to resource constraints. In order to enable this, we propose the following APIs and integration with [Gateway](https://github.com/kubernetes-sigs/gateway-api).


### Should this change be within `ray` or outside?
Within Ray. For better code maintenance.

## Stewardship

### Shepherd of the Proposal
- @kevin85421

## Design and Architecture

### Proposed API

[target_capacity](https://github.com/ray-project/ray/blob/2ae9aa7e3b198ca3dbe5d65f8077e38d537dbe11/python/ray/serve/schema.py#L38) (int [0, 100]) - [Implemented in Ray 2.9](https://github.com/ray-project/ray/commit/86e0bc938989e28ada38faf25b75f717f5c81ed3)
- Definition: (num_current_serve_replicas / num_expected_serve_replicas) * 100%
- Ray Serve / RayService assumes that both the old and new RayClusters have the same capacity to handle requests. Consequently, the new RayCluster can handle up to target_capacity_new of the total requests.

max_surge_percent  (int [0, 100])
- max_surge_percent represents the maximum allowed percentage increase in capacity during a scaling operation. It limits how much the new total capacity can exceed the original target capacity. 
    - The formula is: target_capacity_old + target_capacity_new <= (100% + max_surge_percent)

After adding the above field to the Ray Serve chema, these APIs will be added to the KubeRay CR spec and can be specified by the user as follows:

```sh
type RayServiceUpgradeSpec struct {
  // The target capacity percentage of the upgraded RayCluster.
  // Defaults to 100% target capacity.
  // +kubebuilder:default:=100
  TargetCapacity *int32 `json:"targetCapacity,omitempty"`
  // The maximum percent increase of the total capacity during a scaling operation.
  // Defaults to 100%, i.e. a new RayCluster with equal capacity can be scaled up.
  // +kubebuilder:default:=100
  MaxSurgePercent *int32 `json:"maxSurgePercent,omitempty"`
}
```

The RayService spec would then look as follows:
```sh
// RayServiceSpec defines the desired state of RayService
type RayServiceSpec struct {
 // UpgradeStrategy represents the strategy used when upgrading the RayService. Currently supports `NewCluster` and `None`
  UpgradeStrategy *RayServiceUpgradeStrategy `json:"upgradeStrategy,omitempty"`
  // UpgradeSpec defines the scaling policy used when upgrading the RayService with `NewCluster` strategy.
	UpgradeSpec *RayServiceUpgradeSpec `json:"UpgradeSpec,omitempty"`

	// Defines the applications and deployments to deploy, should be a YAML multi-line scalar string.
	ServeConfigV2  string         `json:"serveConfigV2,omitempty"`
	RayClusterSpec RayClusterSpec `json:"rayClusterConfig,omitempty"`
  ...
}
```

```sh
apiVersion: ray.io/v1
kind: RayService
metadata:
  name: example-rayservice
spec:
  upgradeStrategy: "NewCluster"
  upgradeSpec:
    targetCapacity: 50
    maxSurgePercent: 50
  serveConfigV2: |
    ...
```

#### Key requirements:
- If users want to use the original blue/green deployment, RayService should work regardless of whether the K8s cluster has Gateway-related CRDs
- If users specify both `target_capacity` and `max_surge_percent` Ray should validate whether there is a conflict in the values before proceeding

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

Manual Upgrade:
1. The User specifies both `max_surge_percent` and `target_capacity` in the RayServce `UpgradeSpec`.
   The KubeRay controller validates whether there is a conflict in these values (i.e. the new total `target_capacity` (old + new) must be less than or equal to 100% + `max_surge_percent`).
3. The KubeRay controller creates a Gateway and Routes for the RayService instance, if this is not possible, the controller defaults to a blue/green upgrade strategy.
2. KubeRay creates a new, upgraded RayCluster with `target_capacity` (i.e. with # replicas = `target_capacity`/100 * `total_replicas`).
3. The KubeRay controller switches `target_capacity` * 100% of the requests to the new RayCluster once it is ready.
4. KubeRay scales down the old RayCluster according to the `target capacity` and `max_surge_percent`.
5. Users then increase the `target_capacity` and repeat the above steps until the `target_capacity` reaches 1.

Automatic Upgrade:
1. The user only specifies `max_surge_percent`.
2. The KubeRay controller creates a Gateway and Routes for the RayService instance, if this is not possible, the controller defaults to a blue/green upgrade strategy.
3. The upgraded RayCluster scales up to (1 + `max_surge_percent` - `target_capacity_old`), and the old RayCluster decreases its `target_capacity` until `target_capacity_new` plus `target_capacity_old` equals 1.

## Compatibility, Deprecation, and Migration Plan

### Compatibility

Below is a compatibility matrix for the proposed supported versions of Kubernetes, KubeRay, and Gateway API:

| KubeRay |  Ray   | Kubernetes |  Gateway API  |
| ------- | ------ | ---------- | ------------- |
|  v1.2.2 | 2.8.0+ |   1.23+    | Not Supported |
|  v1.3.0 | 2.9.0+ |   1.25+    | v1.1+ (v1)    |

Gateway API [guarantees support](https://gateway-api.sigs.k8s.io/concepts/versioning/#supported-versions) for a minimum of the 5 most recent Kubernetes minor versions (v1.27+). Gateway still supports Kubernetes v1.24+ but no longer tests against these older minor versions. The latest supported version is v1 as released by the [v1.2.1](https://github.com/kubernetes-sigs/gateway-api/releases/tag/v1.2.1) release of the Gateway API project. For KubeRay we should support a minimum of Kubernetes [v1.25+](https://kubernetes.io/blog/2023/10/31/gateway-api-ga/#cel-validation) to avoid the additional install of a CEL validating webhook.

 
## Test Plan and Acceptance Criteria

We'll add unit and e2e tests in both Ray Core and KubeRay to test against the new API fields and Gateway CRDs. We should also add Kubernetes compatibility tests to ensure the compatibility matrix with Ray, KubeRay, and Gateway API is not broken.
