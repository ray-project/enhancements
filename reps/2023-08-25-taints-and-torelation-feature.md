
## Summary - Taints&Toleration
We plan to introduce the Taints & Tolerations feature to achieve scarce resource isolation, especially for GPU nodes, where preventing ordinary CPU tasks from being scheduled on GPU nodes is a key requirement that many ray users expect.

### General Motivation
The main purpose of this feature is to achieve resource isolation for scarce resources, especially preventing CPU tasks from being scheduled to GPU nodes when no specific scheduling strategy is declared, to avoid occupying GPU node resources. According to our understanding, this requirement is expected by many community users.

**Ray community users’s demand**:  
clarkzinzow:  
[Core] [Hackathon] Affinity and anti-affinity prototype (2/n): Taints and tolerations for inverted anti-affinity. https://github.com/ray-project/ray/pull/21614

Janan Zhu:
![image](https://github.com/ray-project/ray/assets/11072802/d7571d4c-e25d-44fc-ad35-bf6bd8007c74)

Zoltan:
![image](https://github.com/ray-project/ray/assets/11072802/84ab96d4-c68b-4251-95f5-30b3baf29221)

Ray Chinese Community: AntGroup & huawei  
<div align="center">
<img src=https://github.com/ray-project/ray/assets/11072802/f70dda7e-1920-41c6-a24f-9aa5bf21bf16 width=150/>
</div>

### Should this change be within `ray` or outside?
1. The capability of resource isolation for scarce resources needs to be implemented at the Ray Core level. 
2. This enhancement is aimed at improving Ray's scheduling capabilities and better supporting its positioning as an AI infrastructure.

![Ray scheduling framework](https://github.com/ray-project/enhancements/assets/11072802/9c1922d8-8647-44d4-94d3-358bb40a5bee)

## Stewardship
### Required Reviewers
@ericl @stephanie-wang @scv119 (Chen Shen)  @Sang Cho @jjyao (Jiajun Yao) @Yi Cheng @wumuzi520 SenlinZhu @SongGuyang @Chong Li  

### Shepherd of the Proposal (should be a senior committer)
@jjyao (Jiajun Yao)


## Design and Architecture
### 1. Concepts
![Taints & Tolerations concepts](https://github.com/ray-project/enhancements/assets/11072802/24d2f19b-23a0-43e3-8138-01e4233577e0)

1. If you don't want **normal cpu task/actor** to be scheduled on **GPU node**, You can add a taint to a gpu node(node1) using ray cli. For example:
```
# node1
ray start --taints={"gpu_node":"true"} --num-gpus=1
```

2. Normal cpu task/actor/placement group will not be scheduled on GPU node. 
  
The actor/pg will **not be scheudled** onto node1
```
actor = Actor.options(num_cpus=1).remote()

pg = ray.util.placement_group(bundles=[{"CPU": 1}])
```

3. Then you want to schedule gpu task onto gpu node(node1), you can **specify a toleration** for task.  

The actor/pg would be able to scheudled onto node1
```
actor = Actor.options(num_gpus=1, tolerations={"gpu_node": Exists()}).remote()

pg = ray.util.placement_group(bundles=[{"GPU": 1}], tolerations={"gpu_node": Exists()})
```

#### You can also use taints to achieve **node isolation**.

1. If you want to isolate a node with memory pressure so that tasks are not scheduled onto it. You can use **ray taint**:
```
ray taint --node-id {node_id_1} --apend {"memory-pressure":"high"}
```
Then the new task/actor/pg will not be schedule onto node1.


2. You can restore the node once the memory pressure on the node is reduced to a low level.
```
ray taint --node-id {node_id_1} --delete {"memory-pressure":"high"}
```
Then the new task/actor/pg will be able to schedule onto node1.

### 2. API
#### 2.1 Set static taints for nodes  
commond line(ray cli)
```
ray start --taints={"gpu_node":"true"}
```
`ray up` / `kuberay` API is similar.

#### 2.2 Set dynamic taints for nodes:  
commond line(ray cli)
```
# append taints
ray taint --node-id xxxxxxxx --apend {"gpu_node":"true"}
or
ray taint --node-ip xx.xx.xx.xx --apend {"gpu_node":"true"}

# remove taints:
ray taint --node-id xxxxxxxx --remove {"gpu_node":"true"}
or
ray taint --node-ip xx.xx.xx.xx --delete {"gpu_node":"true"}
```

Dashboard RESTful  API:
```
# append taints
POST /nodes/taints/{node_id}
{
  "gpu_node":"true"
}

# delete taints
DELETE /nodes/taints/{node_id}
{
  "gpu_node":"true"
}
```

Dashboard Web Page:  
On the Dashboard page, there is an operations feature for setting taints on nodes.


#### 2.3 Toleration API:
Python Task/Actor api:
```
Actor.options(tolerations={"gpu_node": Exists()})

Task.options(tolerations={"gpu_type": In("A100")})

Actor.options(tolerations={
  "gpu_type": In("A100"),
  "memory-pressure": Exist()}
)

```

Python Placement Group API:
```
pg = ray.util.placement_group(bundles=[{"GPU": 1}], tolerations={"gpu_node": Exists()})
ray.get(pg.ready())
```


### 3. Implementation
#### 3.1 Set taints  
Taints are also a type of node label, so the data structure of the current node labels can be reused.

`absl::flat_hash_map<std::string, std::string> labels`   
change to   
`absl::flat_hash_map<std::string, absl::flat_hash_map<std::string, std::string>> labels`

```
class NodeResources {
  NodeResourceSet total;
  NodeResourceSet available;
  ...
  // Map<label_type, Map<label_key, label_value>>  The key-value labels of this node.
  absl::flat_hash_map<std::string, absl::flat_hash_map<std::string, std::string>> labels;
}
```

The label type only support `custom_labels` and `taints`.  
labels example:
```
Map<label_type, Map<label_key, label_value>>
{
  "custom_labels": {"azone": "azone-1"},
  "taints": {"gpu_node": "true"}
}
```

#### 3.2 Node labels resources synchronization mechanism
Reuse the synchronization mechanism of node resources.


#### 3.3 Use Toleration:
3.3.1 Task/Actor options add `tolerations` parameter.
```
actor = Actor.options(num_gpus=1, tolerations={"gpu_node": Exists()}).remote()
```

`tolerations' Reuse the LabelMatchExpressions data structure of NodeLabelScheduling.  
Pass options from Python to ActorCreationOptions/TaskOptions,  
And pass it on to the core_worker, Then generate rpc::TaskSpec.
```
message TaskSpec {
  ...
  bytes task_id = 6;
  ...
  map<string, double> required_resources = 13;
  ...
  LabelMatchExpressions tolerations = 38;
}
```

3.3.2 Scheduling with `taints` & `tolerations`  
Add `tolerations` of `rpc::LabelMatchExpressions` data structure to ResourceRequest.
```
ClusterTaskManager::ScheduleAndDispatchTasks()
    cluster_resource_scheduler_->GetBestSchedulableNode(
      task_spec.GetRequiredPlacementResources().GetResourceMap(),
      task_spec.GetMessage().scheduling_strategy(),
      ...
      task_spec.GetMessage().tolerations()
    )

ResourceRequest resource_request =
      ResourceMapToResourceRequest(
        task_resources, requires_object_store_memory, tolerations);


class ResourceRequest {
 public:
  ...
 private:
  ResourceSet resources_;
  bool requires_object_store_memory_ = false;
  rpc::LabelMatchExpressions tolerations;
};
```

3.3.3 Adapt the logic of taints and tolerations in the SchedulingPolicy.  
Policy:
* HybridSchedulingPolicy
* RandomSchedulingPolicy
* SpreadSchedulingPolicy
* BundleSchedulingPolicy - PlacementGroup

Adapt code example:
```
bool HybridSchedulingPolicy::IsNodeFeasible(
    const scheduling::NodeID &node_id,
    const NodeFilter &node_filter,
    const NodeResources &node_resources,
    const ResourceRequest &resource_request) const {
  if (!is_node_alive_(node_id)) {
    return false;
  }

  if (!node_resources.IsTaintFeasible(resource_request)) {
    return false;
  }

  return node_resources.IsFeasible(resource_request);
}
```

3.3.4 When scheduling tasks to workers in the local task manager, it verifies once again if the node has any taints and if the request's tolerations are feasible.
```
LocalTaskManager::DispatchScheduledTasksToWorkers()
  bool schedulable = cluster_resource_scheduler_->GetLocalResourceManager().IsLocalNodeTainsFeasible(spec.GetTolerations())
```


### 4. Placement group adaptation
4.1 Python Placement Group API:
```
pg = ray.util.placement_group(bundles=[{"GPU": 1}], tolerations={"gpu_node": Exists()})
ray.get(pg.ready())
```

`PlacementGroupCreationOptions` and `rpc::PlacementGroupSpec` add `tolerations` paramter.

```
// common.proto
message PlacementGroupSpec {
  // ID of the PlacementGroup.
  bytes placement_group_id = 1;
  repeated Bundle bundles = 3;
  ....
  double max_cpu_fraction_per_node = 10;
  LabelMatchExpressions tolerations = 11;
}
```


4.2 Adapt the logic of taints and tolerations in the BundleSchedulingPolicy.



4.3 Adapt the logic of check taints&tolerations in `NodeManager::HandleCommitBundleResources` 


### 5. AutoScaler adaptation
Here, the taints&Toleration will be adapted based on the current autoScaler implementation.

5.1 The autoscaler adds a `taints` field to the node type in the initialized available node types.

``` 
NodeTypeConfigDict = Dict[str, Any]
node_types: Dict[NodeType, NodeTypeConfigDict],
available_node_types = self.config["available_node_types"]
->
{ 
    "4c8g" : {
        "resources": ...,
        "max_workers": ...,
        "labels":{"type":"4c8g", "zone": "usa-1"},
        "taints":{"gpu_node":"true"}
    }
}
```

5.2 Add `tolerations` paramter to the data structure of ResourceDemand in the reply of `GetPendingResourceRequests`.

message ResourceDemand {  
  ...  
  **LabelMatchExpressions tolerations = 6;**  
}  

Note: There is a very critical point here.  
After adding the tolerations, if the `tolerations` of ResourceDemand is different. then a new request requirement will be added. eg:  

```
ResourceDemands: [
    {
        {"CPU":1},
        "num_ready_requests_queued" : 1;
        ....
        "tolerations": {"gpu_node": Exists()}
    },
    {
        {"CPU":1},
        "num_ready_requests_queued" : 2;
        ....
        "tolerations": {}
    }
]

```

```
monitor.py
    -> update_load_metrics
            -> self.gcs_node_resources_stub.GetAllResourceUsage
            -> waiting_bundles, infeasible_bundles = parse_resource_demands(
                    resources_batch_data.resource_load_by_shape)
            -> self.load_metrics.update(waiting_bundles, infeasible_bundles)


GCS:

GcsResourceManager::HandleGetAllResourceUsage(rpc::GetAllResourceUsageReply)
    -> GcsResourceManager::FillAggregateLoad


message GetAllResourceUsageReply {
  GcsStatus status = 1;
  ResourceUsageBatchData resource_usage_data = 2;
  ....
}

message ResourceUsageBatchData {
  // The total resource demand on all nodes included in the batch, sorted by
  // resource shape.
  ResourceLoad resource_load_by_shape = 2;
  ...
}

message ResourceLoad {
  // A list of all resource demands. The resource shape in each demand is
  // unique.
  repeated ResourceDemand resource_demands = 1;
}

// Represents the demand for a particular resource shape.
message ResourceDemand {
  map<string, double> shape = 1;
  ...
  LabelMatchExpressions tolerations = 6;
}
```

5.3 According to the request demand with Toleration information and the `taints` labels of the nodes to be added. Calculate which nodes need to be added.

```
Monitor
    -> StandardAutoscaler.update()
            normal_resource, node_affinity_resource = self.load_metrics.get_resource_demand_vector()
            to_launch = self.resource_demand_scheduler.get_nodes_to_launch(
                normal_resource
            )
```

5.4 AutoScaler policy with tolerations (which decide what node needs to add to the cluster)  

The specific logic for adapting taints and tolerations will be simple, mainly involving excluding nodes that have taints and do not match the tolerations of resources requests.

### 6. Dynamic taints vs. Static taints
I prefer to choose Dynamic Taints.

1. Based on the requirements survey, a large number of users want to use the functionality of dynamic taints. They want to allocate a specific GPU node to a special job at certain times, and then remove the taints after the job is completed, allowing the node to be used by other jobs.
2. The implementation code of dynamic taints can leverage the existing resource passing process, making it easy to implement without disrupting the current framework.
3. Impact on auto scaler: It is analyzed that there is no impact.   
For example, if a taints is applied to a node_id_1 of node type A, and an actor is pending, the auto scaler will scale up the type-A node to keep the actor alive.   This behavior is in line with expectations because dynamic taints only add taints to specific node_ids, preventing normal scheduling for that node.


## Compatibility, Deprecation, and Migration Plan
This is a new scheduling feature that ensures compatibility with all existing scheduling APIs and guarantees that existing code will not be affected.

## Test Plan and Acceptance Criteria
- Unit test new code paths and ensure exist code path are not broken
- Test cases:
  - Set static/dynamic taints for nodes.
  - The scheduling correctness of tolerations with the Default, Spread, and NodeAffinity scheduling policies.
  - The scheduling correctness of tolerations with placement group.
  - The AutoScaler
## (Optional) Follow-on Work

### 1. Automatic Tanints&Tolerations for Rare Resources 
Similar to the `ExtendedResourceToleration admission controller` feature in K8s, we have implemented a RareResourceIsolation feature.

1. If a user wants to isolate GPU as rare resources and prevent CPU tasks from being scheduled on nodes with GPUs,   
they can configure the following when starting the Ray cluster head node: 
```
ray start --head --rare_resource_isolation=gpu
``````

2. When a user starts a GPU node, we automatically add the taints `{"rare_resource.ray.io/gpu": "true"}` to that node.   
This ensures that regular CPU tasks/actors do not be scheduled on this node,  eliminating the need for users to manually set node taints. 
```
ray start --num-gpus=1
```

3. When a user schedules a GPU task or actor, they only need to declare the GPU resource. Internally, we automatically assign the following torelations: 
```
ref = task.options(num_gpus=1).remote() 

# The task will be automatically assigned the torelations: {"rare_resource.ray.io/gpu": Exists()}.
```

4. Users can also declare custom resources as rare resources.  
For example:
```
ray start --head --rare_resource_isolation=gpu,my_rare_resource
```

```
ray start --resources='{"my_rare_resource": 1}'
```

```
ref = task.options(resources={"my_rare_resource": 1}).remote() 
```

