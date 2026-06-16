# REP: Deterministic Label-Based Scheduling with Ray Autoscaler and KubeRay

## Summary
### General Motivation
The Ray autoscaler currently supports interacting with GCS to scale a RayCluster based on labels as part of a `ResourceDemand`. Ray nodes created through autoscaling correctly populate default Ray node labels, which can then be used by the scheduler (when specified by the user through a `label_selector`) to constrain scheduling of Tasks/Actors to the desired nodes.

However, there are challenges with deterministically scaling Kubernetes Pods for Tasks/Actors with user-specified labels and scheduling behavior using the current Ray V2 autoscaler. The primary challenge is that dynamically determining the zone, region, or market type for new Ray Pods is difficult due to various configuration options and cloud-provider-specific labels. Additionally, Downward API values are not known until a Pod is actually running on a physical node.

This proposal explores short-term and long-term solutions to support full label-based autoscaling for Ray users on Kubernetes, enabling a user journey where developers can specify label selectors in the remote decorator of their Ray Tasks/Actors and reliably scale Ray nodes with the desired labels/on a physical node of the requested attributes.

### Should this change be within `ray` or outside?
This change primarily affects the Ray Autoscaler V2 logic within the `ray` repository (specifically the `KubeRayProvider` and `ResourceDemandScheduler`), while leaning on existing Kubernetes and KubeRay capabilities.

## Stewardship
### Required Reviewers
* (TBD)

### Shepherd of the Proposal
* (TBD)

## Design and Architecture
### Current Behavior & Challenges
Currently, to achieve label-based autoscaling, users must explicitly configure multiple worker groups in their `RayCluster` CR with specific `rayStartParams` containing the labels, and corresponding `nodeSelectors` in the Pod template. 

When a user submits a Ray task requiring labels not found on any nodes, the Ray V2 autoscaler stalls if there are no available predefined worker group types that satisfy the required labels. 

Challenges with automatically inferring these labels:
1. **Unclear User Requirements**: Risk of complicating KubeRay code by adding region/zone labels without explicit user demand.
2. **Downward API Limitations**: Values aren't known until the Pod is running.
3. **Non-Standard Labels**: Labels like market type vary across cloud providers (e.g., `cloud.google.com/gke-spot` vs `eks.amazonaws.com/capacityType`).

### Desired User Journey
Users should be able to specify label selectors and seamlessly scale pods on the correct infrastructure:

```python
import ray

@ray.remote(label_selector={"ray.io/availability-zone": "us-west-2a"})
class DataProcessor:
  def process(self, data):
      return f"Processed data on a node in us-west-2a. Data: {data}"

@ray.remote(label_selector={"ray.io/market-type": "spot"})
def batch_job(item):
  return f"Completed batch job {item} on a spot instance."

ray.init()

processor = DataProcessor.remote() 
result = ray.get(processor.process.remote("my-data"))
print(result)

spot_result = ray.get(batch_job.remote(1))
print(spot_result)
```

### Proposed Change

#### Short-Term: Set default labels for Kubernetes nodeSelectors
Scope existing efforts to only set default Ray node labels in the `rayStartParams` based on `nodeSelectors` or `nodeAffinity.RequiredDuringSchedulingIgnoredDuringExecution`. These fields define hard requirements for scheduling. This reduces manual configuration required by the user while fully supporting the static RayCluster use case. If a requested label falls outside the predefined worker groups, it will fail to scale (an enhancement could fail early rather than stall).

#### Long-Term: Refactor Ray Autoscaler V2 to Dynamically Create Worker Groups
Instead of only patching replica counts on pre-defined worker templates, the Ray V2 Autoscaler will dynamically create new worker groups in the `RayCluster` CR with the necessary attributes.

**New Workflow**:
1. **ResourceDemandScheduler** receives a demand with abstract Ray labels. It translates them into concrete Kubernetes `nodeSelectors` (e.g., `{"ray.io/availability-zone": "us-central2-b"} -> {"topology.kubernetes.io/zone": "us-central2-b"}`).
2. **Check Existing Groups**: It checks if an existing `workerGroupSpec` satisfies the resources and node selectors. If so, it scales it using the current replica-patching method.
3. **Generate Dynamic Group**: If no resource demands can be met by existing groups, the `KubeRayProvider` dynamically constructs a new `workerGroupSpec` in memory by merging a suitable base configuration with the required dynamic attributes (like the `nodeSelector`).
4. **Patch RayCluster CR**: The autoscaler sends a PATCH request to the Kubernetes API to add this dynamically generated group to the `spec.workerGroupSpecs` list with `replicas: 1`. KubeRay operator sees the new group and creates the Pods. 
5. **Scale Down**: When scaling down, the autoscaler removes the dynamic group from the list, and KubeRay terminates the corresponding Pods.

### Label Selector Use Cases

| Use Case | Currently Supported | Short-Term Fix | Long-Term Fix |
|---|---|---|---|
| Static RayClusters - all use cases | Yes | Yes | Yes |
| Autoscaling node types with `--labels` set | Yes | Yes | Yes |
| Autoscaling new instances with no `--labels` set in `rayStartParams`, but with `nodeSelectors` set for `workerGroupSpec` | No | Yes | Yes |
| Autoscaling in any given region/zone/market-type, regardless of configured `workerGroupSpec` | No | No | Yes |

### Alternatives Considered
1. **Fail Early**: Check early if `label_selectors` are feasible based on the autoscaling config, and fail early rather than queueing.
2. **Pass all Kubernetes labels as Ray labels**: Add a flag to pass all K8s labels to Ray pods. Complicated by Downward API limitations and race conditions.
3. **Directly create Pods from Autoscaler**: Instead of patching the `RayCluster` CR with new worker groups, the autoscaler could directly call the Kubernetes API to create custom Pods. However, this bypasses KubeRay's management, meaning the autoscaler would need to handle cleanup and min/max constraints manually.
4. **New GKENodeProvider**: A cloud-provider-specific node provider (like GKENodeProvider) that tightly integrates with Kubernetes Engine clients.

### Edge Cases and Limitations

1. **Ambiguity in "Base Template" Selection**:
   If multiple `workerGroupSpecs` satisfy a resource request, the autoscaler needs a deterministic way to pick the base template to clone for the dynamic group.
   **Solution**: The autoscaler should define strict selection criteria (e.g., tightest resource fit or highest priority) and users can optionally add a label like `ray.io/is-dynamic-template-base: true` to mark explicitly clonable worker groups.

2. **K8s Naming Constraints & Pod Name Length**:
   KubeRay generates pod names using `${clusterName}-${groupName}-${workerId}`. Kubernetes enforces a strict 63-character DNS label limit.
   **Solution**: The generated `groupName` for dynamic groups must be aggressively hashed and truncated (e.g., `dyn-<hash>`) to safely avoid K8s naming constraint violations.

3. **Taints, Tolerations, and NodeAffinity**:
   Translating Ray labels directly to `nodeSelectors` does not handle Kubernetes Taints and Tolerations. Adding a `nodeSelector` alone won't schedule the Pod if the underlying K8s node is tainted.
   **Limitation**: This design assumes that if a user wants to target a tainted node pool, their chosen base `workerGroupSpec` *must* already have the correct tolerations configured.

4. **`minReplicas` and `maxReplicas` Behavior**:
   Dynamically created groups must have `minReplicas: 0`, or else KubeRay will refuse to scale them down and they become permanent phantom capacity.
   **Solution**: Dynamically generated worker groups must inherit everything from the base group, except `minReplicas` must be forcefully overridden to `0`. `maxReplicas` should inherit from the base template.

5. **`RayCluster` CR Size Limits (Manifest Bloat)**:
   A Kubernetes CR has an `etcd` size limit of 1.5MB. An unbound list of dynamic `workerGroupSpecs` could crash the operator if hundreds of unique zones are requested simultaneously.
   **Solution**: The autoscaler should enforce a maximum limit on the total number of dynamic worker groups allowed at any given time (e.g., 50). Requests exceeding this limit should fail early.

### Detailed Implementation Path

#### 1. Generating Dynamic `SchedulingNode` Candidates in `ResourceDemandScheduler`
**File:** [`python/ray/autoscaler/v2/scheduler.py`](https://github.com/ray-project/ray/blob/master/python/ray/autoscaler/v2/scheduler.py)

In `_try_schedule`, after attempting to schedule on existing node pools, we need to inspect the remaining infeasible requests. If they contain Kubernetes topology label selectors that caused the scheduling failure on static worker groups, we can synthesize a dynamic `SchedulingNode` based on an existing `NodeTypeConfig` but injected with the required labels.

```python
# In `ResourceDemandScheduler._try_schedule`

# Identify remaining requests that have K8s topology label selectors
dynamic_node_pools = []
for req in requests_to_sched:
    if has_topology_labels(req):
        base_node_type = find_base_node_type_for_resources(req, ctx.get_node_type_configs())
        if base_node_type:
            dynamic_node_type_name = f"dynamic-{base_node_type}-{hash_labels(req)}"
            dynamic_node = SchedulingNode.from_node_config(
                ctx.get_node_type_configs()[base_node_type],
                status=SchedulingNodeStatus.TO_LAUNCH,
                node_kind=NodeKind.WORKER,
            )
            # Inject the desired topology labels
            dynamic_node.labels.update(extract_topology_labels(req))
            dynamic_node.node_type = dynamic_node_type_name
            dynamic_node_pools.append(dynamic_node)

# Run a second pass of `_sched_best_node` with `dynamic_node_pools`
# ...
```

#### 2. Patching the `RayCluster` CR in `KubeRayProvider`
**File:** [`python/ray/autoscaler/v2/instance_manager/cloud_providers/kuberay/cloud_provider.py`](https://github.com/ray-project/ray/blob/master/python/ray/autoscaler/v2/instance_manager/cloud_providers/kuberay/cloud_provider.py)

In `_submit_scale_request`, when we loop through `scale_request.desired_num_workers.items()`, we check if the `node_type` exists in the current `RayCluster` spec using `_worker_group_index`. If it doesn't, it indicates a dynamic group, so we construct a new `workerGroupSpec` and append it via a JSON patch.

```python
# In `KubeRayProvider._submit_scale_request`

for node_type, num_workers in scale_request.desired_num_workers.items():
    group_index = _worker_group_index(raycluster, node_type)
    
    if group_index is None:
        # Dynamic group creation
        if num_workers > 0:
            new_group_spec = self._generate_dynamic_group_spec(node_type, num_workers)
            patch = {
                "op": "add",
                "path": "/spec/workerGroupSpecs/-",
                "value": new_group_spec
            }
            patch_payload.append(patch)
        continue
        
    # Existing logic for static groups
    group_max_replicas = _worker_group_max_replicas(raycluster, group_index)
    # ...
```

Additionally, `_generate_dynamic_group_spec` will extract the base template and inject the corresponding `nodeSelector` based on the parsed dynamic `node_type` metadata. 

When a dynamic group scales down to 0 replicas, we need to completely remove it from the `RayCluster` CR to avoid unbounded manifest bloat over time:

```python
# In `KubeRayProvider._submit_scale_request`
for node_type in scale_request.worker_groups_without_pending_deletes:
    if is_dynamic_group(node_type) and num_workers_for(node_type) == 0:
        group_index = _worker_group_index(raycluster, node_type)
        if group_index is not None:
            patch = {
                "op": "remove",
                "path": f"/spec/workerGroupSpecs/{group_index}"
            }
            patch_payload.append(patch)
```

## Compatibility, Deprecation, and Migration Plan
This change is additive and backwards compatible. Existing `RayCluster` CRs and autoscaling configurations will continue to work. The short-term fix only appends default labels if `--labels` isn't manually specified.

## Test Plan and Acceptance Criteria
- Unit tests verifying the translation from Ray labels to K8s `nodeSelectors`.
- Unit tests verifying the patching logic in `KubeRayProvider` adds and removes dynamic `workerGroupSpecs`.
- E2E Autoscaler tests with KubeRay verifying that tasks with unconfigured zone constraints successfully trigger the creation of a dynamic worker group and pod scheduling.

## (Optional) Follow-on Work
- Expanding dynamic translation logic beyond `topology.kubernetes.io` to a wider range of standard Kubernetes scheduling labels.
