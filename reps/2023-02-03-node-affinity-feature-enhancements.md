## Summary
### General Motivation

The current node affinity feature is relatively simple, it can only support affinity with one node. 
Many general functions cannot be supported.

Current node affinity feature don't support functions:
1. Don't support affinity to a batch of nodes. Current only support affinity to one node.
2. Don't support anti-affinity to a node of a batch of nodes. 
3. Don't support to classify nodes by labels, and then affinity or anti-affinity to a certain type of nodes.
4. Don't support affinity/anti-affinity to nodes by node ip or node name. Current only support node id identification.

This REP is to solve the above problems.


### Should this change be within `ray` or outside?

Yes, this will be a complement to ray core's ability to flexibly schedule actors/tasks/node.

## Stewardship
### Required Reviewers
 
@ericl @stephanie-wang @wumuzi520 SenlinZhu @Chong Li  @scv119 (Chen Shen) @Sang Cho @jjyao (Jiajun Yao) @Yi Cheng
### Shepherd of the Proposal (should be a senior committer)
@scv119 (Chen Shen)

## Design and Architecture

### Brief idea
1. Introduct the concept of the static labels of nodes. Add static labels to nodes(only be set on node start), used to classify nodes. Labels(key, value) = Map<String, String>
2. Improve the expression of NodeAffinity to support affinity\anti-affinity and labels expressions.
3. The actual principle of NodeAffinitySchedulingStratgy is. According to the label matching expression in the strategy, traverse and search for LabelsResource of all nodes. Pick out nodes that meet the expression requirements.


![NodeAffinity Concept](https://user-images.githubusercontent.com/11072802/217133358-19ddc916-bf15-4f69-96e1-9cbd157b0e1d.png)

Scheduling Policy | Label Owner | Label select operator
-- | -- | --
NodeAffinity | Node | in, not_in, exists, does_not_exist

### API Design

**The apis of actors/nodes add labels**  
This interface is already very simple, so we will not set up multiple solutions for everyone to discuss.
```python
#  Node add static labels. 
ray start ... --labels={"location": "dc_1"}

# It can also be set by the environment variable RAY_NODE_LABELS={}.

# Dynamically updating node labels is not implemented now, and will be considered in the second stage.
```

**The apis of the actor-affinity/node-affinity scheduling.**  

**Option 1: Simplify through syntactic sugar**


```python
# Scheduled to a node with a specific IP.
actor_1 = Actor.options(
        scheduling_strategy=node_affinity(label_in(key="node_ip", values=["xxx.xxx.xx.xx"], is_soft=false))
    ).remote()

# Try to schedule to the node with A100/P100 graphics card. If not, schedule to other nodes.
actor_1 = Actor.options(
        scheduling_strategy=node_affinity(label_in("gpu_type", ["A100", "P100"], is_soft=true))
    ).remote()

# Do not schedule to the two nodes whose node id is "xxxxxxx"\"aaaaaaaa".
actor_1 = Actor.options(
        scheduling_strategy=node_affinity(label_not_in("node_id", ["xxxxxxx", "aaaaaaaa"], is_soft=false))
    ).remote()

# Schedule to the node with the key label exist "gpu_type".
actor_1 = Actor.options(
        scheduling_strategy=node_affinity(label_exist("gpu_type"))
    ).remote()

# Don't schedule to the node with the key label exist "gpu_type".
object_ref = Task.options(
        scheduling_strategy=node_affinity(label_does_not_exist("gpu_type", is_soft=false))
    ).remote()

# Multiple label expressions can be filled in at the same time, and the relationship between the expressions is "and". The dispatch must satisfy each expression.
# The actual meaning of this expression is that it must be scheduled to a node with a GPU, and as much as possible to a node with a GPU of the A100 type.
actor_1 = Actor.options(
        scheduling_strategy=node_affinity([
            label_in("gpu_type", ["A100"], true),
            label_exists("gpu_type", false)
        ])
    ).remote()

# Note: This api is old. The old api will still be kept for compatibility with old users.
actor_1 = Actor.options(
    scheduling_strategy=NodeAffinitySchedulingStrategy(
        node_id=ray.get_runtime_context().node_id,
        soft=False,
    )
).remote()

def node_affinity(...):
    ...
    return NodeAffinitySchedulingStrategy(...)

def label_in(key, values, is_soft=false):
    ...
    return LabelMatchExpression(...)

def label_not_in(key, values, is_soft=false):
    ...
    return LabelMatchExpression(...)

def label_exists(key, is_soft=false):
    ...
    return LabelMatchExpression(...)

def label_does_not_exist(key, is_soft=false):
    ...
    return LabelMatchExpression(...)

```

**Option 2: another syntactic sugar** 

Personally, I think this Option is not as good as the above Option 1.  
The label_in(key, values, is_soft) form of option 1 is more understandable and better than the form of ("location", LabelMatchOperator.IN, ["dc_1"], false).
```python
actor_1 = Actor.options(
        scheduling_strategy=NodeAffinity([
            ("node_ip", LabelMatchOperator.IN, ["xxx.xxx.xx.xx"], is_soft=false),
    ).remote()

actor_1 = Actor.options(
        scheduling_strategy=NodeAffinity([
            ("node_id", LabelMatchOperator.NOT_IN, ["xxxxxxx", "aaaaaaaa"], is_soft=true),
    ).remote()

object_ref = Task.options(
        scheduling_strategy=NodeAffinity([
            ("gpu_type", LabelMatchOperator.IN, ["A100"], is_soft=true),
            ("gpu_type", LabelMatchOperator.Exist)).
    ).remote()
```

**Option 3: Java-like form**  

This form is similar to Java's syntax. The downside is that it's a bit complicated.
```python
SchedulingStrategyT = Union[None, str,
                            PlacementGroupSchedulingStrategy,
                            NodeAffinitySchedulingStrategy]

class NodeAffinitySchedulingStrategy:
    def __init__(self, match_expressions: List[LabelMatchExpression]):
        self.match_expressions = match_expressions

class LabelMatchExpression:
    """An expression used to select instance by instance's labels
    Attributes:
        key: the key of label
        operator: IN、NOT_IN、EXISTS、DOES_NOT_EXIST,
                  if EXISTS、DOES_NOT_EXIST, values set []
        values: a list of label value
        soft: ...
    """
    def __init__(self, key: str, operator: LabelMatchOperator,
                 values: List[str], soft: bool):
        self.key = key
        self.operator = operator
        self.values = values
        self.soft = soft

actor_1 = Actor.options(scheduling_strategy=NodeAffinitySchedulingStrategy([
                LabelMatchExpression(
                    "node_ip", LabelMatchOperator.IN, ["xxx.xxx.xx.xx"], False)
            ])).remote()

actor_1 = Actor.options(scheduling_strategy=NodeAffinitySchedulingStrategy([
                LabelMatchExpression(
                    "gpu_type", LabelMatchOperator.IN, ["A100"], True),
                LabelMatchExpression(
                    "gpu_type", LabelMatchOperator.EXISTS, None, False)
            ])).remote()
```

**Option 4: Like sql**  

This solution is not recommended.  
This method needs to parse SQL, and the workload will be much larger.  
And users often write wrong sql when using it.
```python
# NodeAffinity use case
actor_1 = Actor.options(
        scheduling_strategy=NodeAffinity("gpu_type in [A100, T100]")
    ).remote()
```

### Example

**1. Affinity to the node of the specified Instance.**  
```
# m6in.2xlarge 8C32G instance node
ray start ... --labels={"instance": "m6in.2xlarge"}

# m6in.large 2C8G instance node
ray start ... --labels={"instance": "m6in.large"}

actor_1 = Actor.options(
        scheduling_strategy=node_affinity(label_in(key="instance", values=["m6in.large"], is_soft=false))
    ).remote()

```

**2. Anti-Affinity to the nodes of the special node_id.**  
```
actor_1 = Actor.options(
        scheduling_strategy=node_affinity(label_not_in(key="node_id", values=["xxxxx", "aaaaa"], is_soft=false))
    ).remote()
```

**3. Anti-Affinity to the node which have GPU.**  
```
actor_1 = Actor.options(
        scheduling_strategy=node_affinity(label_does_not_exist(key="gpu_type", is_soft=false))
    ).remote()
```

**4. It must be scheduled to a node whose region is china, and as much as possible to a node whose availability zone is china-east-1.**  

```
actor_1 = Actor.options(
        scheduling_strategy=node_affinity([
            label_in("region", ["china"], is_soft=false),
            label_in("zone", ["china-east-1"], is_soft=true)
        ])
    ).remote()
```
**5. NodeAffinity can be used together with the CustomResource mechanism.**  
These two mechanisms are completely non-conflicting.  
```
actor_1 = Actor.options(
                num_gpus=1,
                resources={"custom_resource": 1},
                scheduling_strategy=node_affinity([
                    label_exist("gpu_type", is_soft=false),
                    label_in("gpu_type", ["A100", "P100"], is_soft=true),
                ]).remote()
```

**FAQ:**  
This node static labels seems to be very similar to custom_resources. Why not use custom_resources to instance of node_affinity?

1. The semantics of static labels and custom_resources are completely different. custom_resources is a countable resource for nodes. And labels is an attribute or identifier of a node.
2. Now custom_resources cannot implement the function of anti-affinity.
3. Using labels to classify nodes and implement affinity/anti-affinity will be more natural and easier for users to accept.

### Implementation plan

![NodeAffinity Concept](https://user-images.githubusercontent.com/11072802/217133358-19ddc916-bf15-4f69-96e1-9cbd157b0e1d.png)

1. Add the labels to node info
```
message GcsNodeInfo {
  // The ID of node.
  bytes node_id = 1;
  ...
  // The user-provided identifier or name for this node.
  string node_name = 12;
  // The static labels of nodes.
  map<String, String> labels = 13;
}
```
2. Add the labels data structure to the resource synchronization data structure(NodeResources). 
```
NodeResources {
  ResourceRequest total;
  ResourceRequest available;
  /// Only used by light resource report.
  ResourceRequest load;
  /// Resources owned by normal tasks.
  ResourceRequest normal_task_resources
  /// Map<string, string> Nodes labels information
  absl::flat_hash_map<string, string> labels;
}
```

3. When a node registers with GCS and broadcasts to all nodes, add the static labels information to NodeResources.  

Because NodeInfo originally had a mechanism to synchronize to the entire cluster. So there is no need to modify too much code.

4. Scheduling optimization through Labels 

Now any node raylet has node static labels information for all nodes.  
when NodeAffinity schedules, if it traverses the Labels of each node, the algorithm complexity is very large, and the performance will be poor.   
<b> Therefore, it is necessary to generate a full-cluster node labels index table to improve scheduling performance. </b>

The label index table is only used to obtain which nodes have this key and value. The complexity of this algorithm is O(1).
The matching logic of the expression is as follows:
> label_in(key, values[]) = LabelManager.GetNodesByKeyAndValue()

> label_not_in(key, values[]) = All_Nodes - LabelManager.GetNodesByKeyAndValue()

```
class ClusterLabelManager {
 public:
  absl::flat_hash_set<NodeID> GetNodesByNodeKeyAndValue(const std::string &key, const absl::flat_hash_set<std::string> &values) const;

  absl::flat_hash_set<NodeID> GetNodesByNodeKey(const std::string &key) const;

  void AddNodeLabels(const std::shared_ptr<NodeInfo> &node);

  void RemoveNodeLabels(const std::shared_ptr<NodeInfo> &node);

 private:
 <label_key, <lable_value, <node_id, ref_count>>>> labels_to_nodes_;
}
```

### If the Match Expression Cannot be satisfied
If the matching expression cannot be satisfied, The actor/task will be add to the pending queue. Util the matching expression all be statisfied.

1. resources are enough, but node_id of affinity don't exist. -> throw exception, compatible with current behavior 

2. resources are enough, but labels affinity  cannot be satisfied? -> Hang & report schedule failed event and detail unstaisfed reason to exposed to users

3. labels affinity can be satisfied, but resources are not enough? -> Hang & report schedule failed event and detail unstaisfed reason to exposed to users

4. both labels affinity and resources are not satisfied? -> Hang & report schedule failed event and detail unstaisfed reason to exposed to users


### how other system achieve the same goal?
1、K8s
This solution is to learn the PodAffinity/NodeAffinity features of K8s。
https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity

Scheduling Policy | Label Owner | Operator
-- | -- | --
nodeAffinity | NODE | In, NotIn, Exists, DoesNotExist, Gt, Lt
podAffinity | POD | In, NotIn, Exists, DoesNotExist
podAntiAffinity | POD | In, NotIn, Exists, DoesNotExist


### what's the alternative to achieve the same goal?
** Option 3: putting Labels into the custom resource solution  **  
```
class ResourceRequest {
    absl::flat_hash_map<ResourceID, FixedPoint> resources_;
}

Add Actor/Task/Node labels to resources_ as custom resources.
eg:  
{
    "CPU": 16,
    "memory": xx,
    "custom_resources": xx,
    "actor_labels_key@value": 1,
    "task_labels_key@value": 1,
    "node_labels_key@value": 1,
}
```
If you put labels into custom_resources, you need to do the following adaptation:
1. Compared with custom_resources, labels need to add a specific prefix to distinguish them from custom_resources.
2. The key and value of Labels need to be concatenated with special characters (@).
3. When using Labels to build a Labels index table, you need to parse the resources key.

**DisAdvantages:**
1. Labels unlike cpu resource these are numeric types. Compared with the above scheme. This will destroy the concept of coustrom resouce.
2. Actor and Task are isolated by namespace. It is difficult to isolate through namespace if adding custom_resource.
2. The Label index table of all nodes can be constructed from the ActorLabels information of each node. If you use Custom Resource, this requires parsing the resource_key and doing a lot of string splitting which will cost performance.
3. If custom_resource happens to be the same as the spliced string of labels. Then it will affect the correctness of scheduling.

### AutoScaler adaptation
Here, the Node Affinity scheduling will be adapted based on the current autoScaler implementation.

1. The autoscaler adds a labels field to the node in the initialized available node types.

``` 
# e.g., {"resources": ..., "max_workers": ...}.
NodeTypeConfigDict = Dict[str, Any]
node_types: Dict[NodeType, NodeTypeConfigDict],
available_node_types = self.config["available_node_types"]
->
{ 
    "4c8g" : {
        "resources": ...,
        "max_workers": ...,
        "labels":{"type":"4c8g", "zone": "usa-1"}
    }
}
```

2. AutoScaler's Monitor requests Gcs to add schedule_stategy to the reply data structure of ResourceDemand

message ResourceDemand {  
  ...  
  **SchedulingStrategy scheduling_strategy = 5;**  
}  

Note: There is a very critical point here.  
After adding the scheduling policy, if the NodeAffinity policy in the scheduling policy is different. then a new request requirement will be added. eg:  

```
ResourceDemands: [
    {
        {"CPU":1},
        "num_ready_requests_queued" : 1;
        ....
        "scheduling_strategy": node_affinity(label_exist("has_gpu"))
    },
    {
        {"CPU":1},
        "num_ready_requests_queued" : 2;
        ....
        "scheduling_strategy": node_affinity(label_in("has_gpu", "false"))
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
  // The resource shape requested. This is a map from the resource string
  // (e.g., "CPU") to the amount requested.
  map<string, double> shape = 1;
  // The number of requests that are ready to run (i.e., dependencies have been
  // fulfilled), but that are waiting for resources.
  uint64 num_ready_requests_queued = 2;
  // The number of requests for which there is no node that is a superset of
  // the requested resource shape.
  uint64 num_infeasible_requests_queued = 3;
  // The number of requests of this shape still queued in CoreWorkers that this
  // raylet knows about.
  int64 backlog_size = 4;

  SchedulingStrategy scheduling_strategy = 5;
}
```

3. Divide the received RequestDemands into normal Request and NodeAffinity Request
```
normal_resource, node_affinity_resource = self.load_metrics.get_resource_demand_vector()

node_affinity_resource: List[AffintyResouces]
AffintyResouces: {"resources": ResourceDict , "affinity_expressions": AffinityExpressions}
```

4. According to the request demand with NodeAffinity information and the static labels of the nodes to be added. Calculate which nodes need to be added.

```
Monitor
    -> StandardAutoscaler.update()
            normal_resource, node_affinity_resource = self.load_metrics.get_resource_demand_vector()
            to_launch = self.resource_demand_scheduler.get_nodes_to_launch(
                normal_resource,
                node_affinity_resource
            )

self.resource_demand_scheduler.get_nodes_to_launch:
    -> get_node_affinity_request_need_node(node_affinity_resource)
    -> _add_min_workers_nodes()

The implementation logic in get_node_affinity_request_need_node will be more complicated. The main idea is to traverse whether the static labels to be added meet the NodeAffinity policy of requestDemand. If it is satisfied, the master will be selected to join according to the scoring situation.

Although it will be more complicated here, they are all independent modules and will not have a great impact on the current implementation structure of autoScaler.
```

5. AutoScaler policy with node affinity (which decide what node needs to add to the cluster)  

Here is just a general expression of the solution, and the specific strategy needs to be designed in detail.

This piece is indeed more complicated, and can be realized as follows:
>The types of nodes that AutoScaler can add to the cluster are generally predicted or configured. For example:
Node Type 1: 4C8G labels={"instance":"4C8G"} ,
Node Type 2: 8C16G labels={"instance":"8C16G"},
Node Type 3: 16C32G labels={"instance":"16C32G "}

> The label of NodeAffinity is the case in the standby node.
For example, the following scenario:
Actor.options(num_cpus=1, scheuling_strategy=NodeAffinity(label_in("instance": "4C8G")).remote()
If the Actor is pending, the autoscaler traverses the prepared nodes to see which node meets the requirements of [resource: 1C, node label: {"instance":"4C8G"}]. If so, add it to the cluster.

> The label of NodeAffinity is a unique special label
In this scenario, it is considered that the Actor/Task wants to be compatible with a special node, and there is no need to expand the node for it.
eg:
Actor.options(num_cpus=1, scheuling_strategy=NodeAffinity(label_in("node_ip": "xxx.xx.xx.xx")).remote()

> anti-affinity to a node with special label.  
It can be pre-judged whether the prefabricated nodes can meet this anti-affinity requirement, and if so, they can be added to the cluster.

> Soft strategy.
In this scenario, it can be pre-judged whether the labels and resources of the prefabricated nodes can be satisfied, and if so, use such nodes. If none of the labels can satisfy, just add a node with enough resources.


**Nodes:**   
Now Ray has more and more scheduling strategies. If AutoScaler still simulates the scheduling again according to the current implementation method, then code maintenance will become more and more troublesome. For each additional scheduling strategy, AutoScaler needs to be adapted once, which is obviously unreasonable. So I think AutoScaler is still in urgent need of refactoring.

## Compatibility, Deprecation, and Migration Plan

## Test Plan and Acceptance Criteria
All APIs will be fully unit tested. All specifications in this documentation will be thoroughly tested at the unit-test level. The end-to-end flow will be tested within CI tests. Before the beta release, we will add large-scale testing to precisely understand scalability limitations and performance degradation in large clusters. 

## (Optional) Follow-on Work

### 1. Expression of "OR" semantics.
Later, if necessary, you can extend the semantics of "OR" by adding "is_or_semantics" to ActorAffinitySchedulingStrategy.
```
class NodeAffinitySchedulingStrategy:
    def __init__(self, match_expressions: List[LabelMatchExpression], is_or_semantics = false):
        self.match_expressions = match_expressions
        self.is_or_semantics = 
```

### 2. Support dynamic updating of node labels.
The current node label is static (only set when the node is started), and then the node label is dynamically updated through the interface.

### 3. Use node labels to achieve Runtime Env acceleration
Now sometimes the initialization is slow because of the many dependencies of the Runtime env. If the same runtime env is scheduled on the same node, then some dependency caches can be reused, which can improve the deployment speed of the runtime env.

Using the Node labels feature can naturally realize that actors/tasks with the same runtime env are scheduled to the same node as much as possible.

1. Support dynamic updating of node labels.

2. The Runtime Env config add a field of {"use_node_labels_to_acceleration": true}.

3. When the first actor/task using Runtime Env is dispatched to a node, and has {"use_node_labels_to_acceleration": true} configured. Then, according to the configuration of Runtime Env, generate the identity hash value, and put a label on this node: Node.update_labels("exist_runtime_env", "["runtime_env_id_1"]"). In this way, the module responsible for scheduling knows that the runtime env "runtime_env_id_1" has been deployed on this node

4. When the second Actor/Task that uses the same runtime env is scheduled, a section can be added to the hybrid scheduling strategy to schedule to nodes with the same runtime env as much as possible:   
a. By traversing the labels{"exist_runtime_env": "..."} of each node, query which nodes have the same runtime_env.   
b. If a node with the same runtime env is found, the weight of this node will be increased.