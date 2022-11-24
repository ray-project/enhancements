
## Summary
### General Motivation

Introduce the Labels mechanism. Give Labels to Actors/Tasks/Nodes/Objects. 
Affinity features such as ActorAffinity/TaskAffinity/NodeAffinity can be realized through Labels.


### Should this change be within `ray` or outside?

Yes, this will be a complement to ray core's ability to flexibly schedule actors/tasks/node.

## Stewardship
### Required Reviewers
 
@wumuzi520 SenlinZhu @Chong Li  @scv119 (Chen Shen) @jjyao (Jiajun Yao) @Yi Cheng
### Shepherd of the Proposal (should be a senior committer)


## Design and Architecture

### Brief idea
1. Introduce the concept of Label. Add the Labels attribute to Actor/Task/Node. Labels = Map<String, String>.
2. After Actor/Task are scheduled to a certain node, the Labels of Actor/Task will be attached to the node resource(Named: LabelsResource). Node's Labels are naturally in the node resource.
3. Actor/Task scheduling can choose Actor/Task/NodeAffinitySchedulingStratgy. 
4. The actual principle of Actor/Task/NodeAffinitySchedulingStratgy is. According to the label matching expression in the strategy, traverse and search for LabelsResource of all nodes. Pick out nodes that meet the expression requirements.

![LabelsAffinity](https://user-images.githubusercontent.com/11072802/203686866-385235b5-e08b-4aac-9c31-512621129bd4.png)

### API Design
```python
# Actor add labels.
actor_1 = Actor.options(labels={
    "location": "dc_1"
}).remote()

# Task add labels.
task_1 = Task.options(labels={
    "location": "dc_1"
}).remote()

#  Node add static labels.
ray start ... --labels={"location": "dc_"}

```python
SchedulingStrategyT = Union[None, str,
                            PlacementGroupSchedulingStrategy,
                            ActorAffinitySchedulingStrategy,
                            TaskAffinitySchedulingStrategy,
                            NodeAffinitySchedulingStrategy]

class ActorAffinitySchedulingStrategy:
    def __init__(self, match_expressions: List[LabelMatchExpression]):
        self.match_expressions = match_expressions

class TaskAffinitySchedulingStrategy:
    def __init__(self, match_expressions: List[LabelMatchExpression]):
        self.match_expressions = match_expressions

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

# ActorAffinity use case
actor_1 = Actor.options(scheduling_strategy=ActorAffinitySchedulingStrategy([
                LabelMatchExpression(
                    "location", LabelMatchOperator.IN, ["dc_1"], False)
            ])).remote()
```


### Implementation plan

![LabelsAffinity-资源调度](https://user-images.githubusercontent.com/11072802/203686851-894536d0-86fa-4fab-a5fe-2e656ac198e5.png)

1. Add the actor_labels data structure to the resource synchronization data structure(ResourcesData and NodeResources). 
```
message ResourcesData {
  // Node id.
  bytes node_id = 1;
  // Resource capacity currently available on this node manager.
  map<string, double> resources_available = 2;
  // Indicates whether available resources is changed. Only used when light
  // heartbeat enabled.
  bool resources_available_changed = 3;

  // Map<label_type, Map<namespace, Map<label_key, label_value>>> Actors/Tasks/Nodes labels information
  repeat Map<string, Map<string, Map<string, string>>> labels = 15
  // Whether the actors of this node is changed.
  bool labels_changed = 16,
}


NodeResources {
  ResourceRequest total;
  ResourceRequest available;
  /// Only used by light resource report.
  ResourceRequest load;
  /// Resources owned by normal tasks.
  ResourceRequest normal_task_resources
  /// Map<label_type, Map<namespace, Map<label_key, label_value>>> Actors/Tasks/Nodes labels information
  absl::flat_hash_map<string, absl::flat_hash_map<string, absl::flat_hash_map<string, string>>> actor_labels;
}
```

2. Adapts where ResourcesData is constructed and used in the resource synchronization mechanism.  
a. NodeManager::HandleRequestResourceReport  
b. NodeManager::HandleUpdateResourceUsage 

3. Add Labels of actors/tasks to NodeResources during Actors/tasks scheduling

4. Scheduling optimization through Labels  
Take ActorAffinity's scheduling optimization scheme as an example：  
Now any node raylet has ActorLabels information for all nodes. 
However, when ActorAffinity schedules, if it traverses the Labels of all Actors of each node, the algorithm complexity is very large, and the performance will be poor.   
<b> Therefore, it is necessary to generate a full-cluster ActorLabels index table to improve scheduling performance. <b>

```
class GcsLabelManager {
 public:
  absl::flat_hash_set<NodeID> GetNodesByKeyAndValue(const std::string &ray_namespace,
      const std::string &key, const absl::flat_hash_set<std::string> &values) const;

  absl::flat_hash_set<NodeID> GetNodesByKey(const std::string &ray_namespace,
                                            const std::string &key) const;

  void AddActorLabels(const std::shared_ptr<GcsActor> &actor);

  void RemoveActorLabels(const std::shared_ptr<GcsActor> &actor);

 private:
 <namespace, <label_key, <lable_value, [node_id]>>> labels_to_nodes_;
 <node_id, <namespace, [actor]>>  nodes_to_actors_;
}
```

Actor scheduling flowchart：
![Actor scheduling flowchart](https://user-images.githubusercontent.com/11072802/202128385-f72609c5-308d-4210-84ff-bf3ba6df381c.png)

Node Resources synchronization mechanism:
![Node Resources synchronization mechanism](https://user-images.githubusercontent.com/11072802/203783157-fad67f25-b046-49ac-b201-b54942073823.png)

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
Option 2: Use LabelAffinitySchedulingStrategy instead of ActorAffinitySchedulingStrategy/TaskAffinitySchedulingStrategy/NodeAffinitySchedulingStrategy
Some people think that ActorAffinity/TaskAffinity is dispatched to the Node corresponding to the actor/Task with these labels.
Why not assign both ActorLabels and TaskLabels to Node?   
Then the scheduling API only needs to use the LabelAffinitySchedulingStrategy set of APIs to instead of ActorAffinitySchedulingStrategy/TaskAffinitySchedulingStrategy/NodeAffinitySchedulingStrategy.

API Design:
```
SchedulingStrategyT = Union[None, str,
                            PlacementGroupSchedulingStrategy,
                            LabelAffinitySchedulingStrategy]

class LabelAffinitySchedulingStrategy:
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


actor_1 = Actor.options(scheduling_strategy=LabelAffinitySchedulingStrategy([
                LabelMatchExpression(
                    "location", LabelMatchOperator.IN, ["dc_1"], False)
            ])).remote()
```

Why?
1. Actor and Task are isolated by namespace. But Node has no namespace isolation. As a result, Actor/Task labels cannot be directly integrated with Node labels.
2. Separately divided into Actor/Task/NodeAffinity. In this way, the interface semantics will be more simple and concise. Avoid conceptual confusion.
3. LabelAffinity combines the labels of Actor/Task/Node at the same time. The final scheduling result may not be accurate. Users only want to use ActorAffinity. But other Tasks have the same labels with actor labels. This scene will make scheduling result not be accurate.

Advantages:
1. It is possible to realize the scene of affinity with Actor/Task/Node at the same time. for example:
The user wants to affinity schedule to <b> some Actors and nodes in a special computer room. <b>
However, according to the results of internal user research, most of the requirements are just to realize Actor/Task "collocate" scheduling or spread scheduling. So using a single ActorAffinity/TaskAffinity/NodeAffinity can already achieve practical effects.  

And the same effect can be achieved by combining the option 1 with custom resources

## Compatibility, Deprecation, and Migration Plan

## Test Plan and Acceptance Criteria
All APIs will be fully unit tested. All specifications in this documentation will be thoroughly tested at the unit-test level. The end-to-end flow will be tested within CI tests. Before the beta release, we will add large-scale testing to precisely understand scalability limitations and performance degradation in large clusters. 

## (Optional) Follow-on Work

### Expression of "OR" semantics.
Later, if necessary, you can extend the semantics of "OR" by adding "is_or_semantics" to ActorAffinitySchedulingStrategy.
```
class ActorAffinitySchedulingStrategy:
    def __init__(self, match_expressions: List[LabelMatchExpression], is_or_semantics = false):
        self.match_expressions = match_expressions
        self.is_or_semantics = 
```

### ObjectAffinitySchedulingStrategy
If the user has a request, you can consider adding the attributes of labels to objects. Then the strategy of ObjectAffinity can be launched。