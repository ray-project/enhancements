## Summary
### General Motivation

This REP summarizes the current state of the node label scheduling feature enhancement and the remaining work to fully support scheduling using label selectors in Ray.

### Should this change be within `ray` or outside?

The change should be within Ray since it's a direct enhancement to the Ray scheduler.

## Stewardship
### Required Reviewers
@MengjinYan

### Shepherd of the Proposal (should be a senior committer)
@edoakes

## Design and Architecture

### Current implementation state

Ray currently supports passing labels to a node through `ray start` with the `--labels` flag in Python, parsing labels from a json string with `parse_node_labels_json`, and associates a list of labels with a Node through the `_labels` field. Node information, including labels, are saved in the `GcsNodeInfo` data struct when a node is added. Ray also supports setting default labels on node add, but currently only sets `ray.io/node-id`.

To pass labels to a Ray node:
```sh
ray start --head --labels='{"gpu_type": "A100", "region": "us"}'
```

To access node labels:
```python
ray.nodes()[0]["Labels"] == {"gpu_type": "A100", "region": "us"}
```

To schedule nodes based on these labels, users specify `scheduling_strategy=NodeLabelSchedulingStrategy` as follows:
```python
 actor = MyActor.options(
    scheduling_strategy=NodeLabelSchedulingStrategy({"ray.io/availability-zone": In("us-central2-b")})
).remote()
```

With both hard and soft constraints:
```python
MyActor.options(
    actor = MyActor.options(
        scheduling_strategy=NodeLabelSchedulingStrategy(
            {"gpu_type": NotIn("A100", "T100"), "other_key": DoesNotExist()}
            hard={"gpu_type": DoesNotExist()},
            soft={"gpu_type": In("A100")},
        )
    )
).remote()
```

These API are currently [hidden](https://github.com/ray-project/ray/blob/da092abe3d4adfe2c5d94bde64c97a994a2e061b/python/ray/scripts/scripts.py#L628) and not publicly exposed.
The above API is supported through the following internal implementation:

NodeInfo struct:
```python
message GcsNodeInfo {
  ...
  // The key-value labels of this node.
  map<string, string> labels = 26;
  ...
}
```

Add labels from GCS when a Node is added:
```python
void NodeManager::NodeAdded(const GcsNodeInfo &node_info) {
  ...
  // Set node labels when node added.
  absl::flat_hash_map<std::string, std::string> labels(node_info.labels().begin(),
                                                       node_info.labels().end());
  cluster_resource_scheduler_->GetClusterResourceManager().SetNodeLabels(
      scheduling::NodeID(node_id.Binary()), labels);
  ...
}
```

Add default labels:
```python
void NodeManagerConfig::AddDefaultLabels(const std::string &self_node_id) {
  # Adds the default `ray.io/node-id` label to the label mapping
}
```

Get node labels from GCS:
```python
std::unordered_map<std::string, std::string> PythonGetNodeLabels(const rpc::GcsNodeInfo &node_info) {
  # Returns the current list of labels from the GcsNodeInfo
}
```

And finally a `NodeLabelSchedulingStrategy` Scheduling Policy with the following key functions. This scheduling strategy has not yet been added to the [`SchedulingStrategy` proto](https://github.com/larrylian/ray/blob/66c05338b07f1ef149928d4742b5f70c6c49b138/src/ray/protobuf/common.proto#L72), but an alpha version is public in the [Python worker](https://github.com/ray-project/ray/blob/07cdfec1fd9b63559cb1d47b5197ef5318f4d43e/python/ray/util/scheduling_strategies.py#L40).
```python
scheduling::NodeID NodeLabelSchedulingPolicy::Schedule(...) {
    # Filters the feasible nodes - those that satisfy the provided resource request - by the
    # hard constraints of the label selectors and conditions, and then creates another list
    # of those nodes which satisfy both the hard and soft label conditions. Schedule then returns
    # the best node from these two lists.
}

scheduling::NodeID NodeLabelSchedulingPolicy::SelectBestNode(...) {
    # If non-empty, returns a random node from the list of available nodes which satisfy both
    # hard and soft constraints. Else, returns a random node from the list of available nodes which
    # satify the hard conditions. If there are no available nodes, returns a random feasible node
    # from the hard and soft matches, or the hard matches if the former is empty.
}

NodeLabelSchedulingPolicy::FilterNodesByLabelMatchExpressions(...) {
    # Iterates through candidate nodes and returns list of those which satisfy the conditions.
}

NodeLabelSchedulingPolicy::IsNodeMatchLabelExpression(
    const Node &node, const rpc::LabelMatchExpression &expression) const {
    # Returns a bool based on whether a node's labels satisfy the given condition.
    # Supports exists, not exists, in, and not in conditions. We should also extend 
    # support to equal and not equal.
}
```

### Brief idea
In order to implement full label based scheduling as described in the [public proposal](https://docs.google.com/document/d/1DKhPuZERlLbsU4TIQZZ6POCsm1pVMBgN_yn5r0OmaDI), there are several required changes to the existing API and internal implementation in Ray core. Since most of the core functionality for adding, storing, and retrieving node labels is already implemented, the primary changes proposed here are to update the APIs, support autoscaling, and directly schedule nodes based on label selectors when provided, rather than passing a separate scheduling policy.


### API Design

To pass labels to a Ray node, we will amend the `--labels` argument to `ray start` to accept a string list of key-value pairs. Currently the labels argument accepts a json struct.
```sh
ray start --labels "key1=val1,key2=val2"
```

We will also support sourcing labels from a file using bash:
```sh
ray start --labels $(cat labels.txt)
```

To then schedule based on these labels, we will support passing a `label_selector` argument to the `@ray.remote` decorator.
```python
@ray.remote(label_selector={"ray.io/accelerator-type": "nvidia-h100"})
class Actor:
    pass
...

@ray.remote(label_selector={"ray.io/market-type": "spot"})
def my_task():
    pass
```

To schedule placement groups based on labels we will implement support for applying label selectors to placement groups on a per-bundle level. This would require adding a `bundle_label_selector` to the `ray.util.placement_group` constructor.
```python
# Same labels on all bundles
ray.util.placement_group(
    bundles=[{"GPU": 1}, {"GPU": 1}],
    bundle_label_selector=[{"ray.io/availability-zone": "us-west4-a"}] * 2,
)

# Different bundles requiring different labels
ray.util.placement_group(
    bundles=[{"CPU": 1}] + [{"GPU": 1} * 2],
    bundle_label_selector=[{"ray.io/market_type": "spot"}] + [{"ray.io/accelerator-type": "nvidia-h100"} * 2]
)
```

Finally, we will implement a `fallback_strategy` API to support soft constraints or multiple deployment options if the initial `label_selector` cannot be satisfied.
```python
@ray.remote(
    label_selector={"instance_type": "m5.16xlarge"},
    fallback_strategy=[
        # Fall back to an empty set of labels (no constraints).
        # This is equivalent to a "soft" constraint for an m5.16xlarge.
        {"label_selector": {}},
    ],
)
```

### Implementation plan

A portion of the internal implementation to save node labels, match based on label conditions, and support node labels in the core Python worker already exists. The primary changes required are to update the current APIs to those described above, move the logic from the `NodeLabelSchedulingStrategy` directly to the [cluster resource scheduler](https://github.com/ray-project/ray/blob/07cdfec1fd9b63559cb1d47b5197ef5318f4d43e/src/ray/raylet/scheduling/cluster_resource_scheduler.cc#L149), and implement support for autoscaling.

Overview of Ray scheduler steps during label based scheduling:
1. Ray gets a request to schedule nodes based on some resources and labels.
2. Ray filters the feasible nodes by those that satisfy the resource request. A feasible node is one with sufficient total resources to satisfy the request, although those resources may not currently be available.  
3. Ray hard matches nodes that satisfy the resource request with those that satisfy the label selector and expression.
4. If no nodes match and a `fallback_strategy` is provided, filter by the provided fallback label selector and return the list of candidate nodes.
5. Ray returns the best schedulable node from the list of available (or feasible if no nodes are available) that satisfy the expressions in steps 3 and/or 4.

Remaining steps to implement the label based scheduling feature:
- [ ] (P1) Update `--labels` argument to take either a list of strings or read from file and expose this API publicly
- [ ] (P1) Add `label_selector` API to `@ray.remote` decorator to schedule tasks/actors
- [ ] (P1) Add `bundle_label_selector` to the `ray.util.placement_group` constructor to apply a set of `label_selector`s to placement group bundles
- [x] (P1) Ray saves label info associated with a node in `GcsNodeInfo` - already implemented
- [ ] (P1) Update `ClusterResourceScheduler::GetBestSchedulableNode` to enforce `label_selector` conditions when returning list of candidate nodes. This will eventually replace `SchedulingOptions::NodeLabelScheduling(scheduling_strategy)`.
- [ ] (P1) Populate list of default labels automatically, currently only supports `ray.io/node-id`
- [ ] (P1) Adapt Ray V2 Autoscaler to parse labels from K8s Pod Spec and generate a `--labels` arg to `rayStartParams`
- [ ] (P1) Update Autoscaler data model to pass label information by adding a labels field to the `ResourceRequest` message
- [ ] (P1) Update Autoscaler bin packing logic to directly consider label matching
- [ ] (P2) Determine whitelist of K8s labels to always pass to Ray nodes
- [ ] (P2) Implement `fallback_strategy` API to match available/feasible nodes by the provided conditions if `label_selector` returns 0 matches
- [ ] (P2) Update documentation/examples to use updated `label_selector` API
- [ ] (P2) Update library usage of `NodeLabelSchedulingStrategy` with `label_selector` API
- [ ] (P2) Add labels argument to `request_resource()` SDK function used by Ray libraries

### Autoscaler adaptation
Label based scheduling support should be added to the Ray V2 Autoscaler, only supporting the Kubernetes stack at first. Once the VM stack is also migrated to the V2 autoscaler, we can extend label based scheduling support. In order to inform scaling decisions based on user provided label selectors to Ray tasks/actors, it's necessary to propogate label information at runtime to the autoscaler and GCS. The required changes to the Ray Autoscaler V2 APIs and data model are described above in the implementation plan.


## Compatibility, Deprecation, and Migration Plan
As the above APIs are implemented, we can deprecate redundant functionality like `accelerator-type`, but retain NodeAffinitySchedulingStrategy until soft constraints are supported through `fallback_strategy`. We will update libraries, documentation, and examples where appropriate to use the new label selector API.


## Test Plan and Acceptance Criteria
All APIs will be rigorously unit tested, ensuring thorough validation of the documented specifications. End-to-end flows will be covered in CI tests. Prior to promoting this API to beta, we will add large-scale tests to assess scalability limits and performance impact on large clusters. End-to-end testing can be added to the KubeRay repo for the K8s stack as well as to the Ray V2 Autoscaler as part of that feature's promotion to beta.
