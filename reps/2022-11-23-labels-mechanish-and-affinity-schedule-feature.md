## Summary
### General Motivation

Introduce labels mechanism, which associate an enumerated property to Ray nodes. We can static assign labels to Ray node on start up, or dynamically change node's label through Ray scheduling APIs.

Labels mechanism makes it easy to implement Actor and Node affinity.(Taints and Tolerations style)


### Should this change be within `ray` or outside?

Yes, this will be a complement to ray core's ability to flexibly schedule actors/tasks/node.

## Stewardship
### Required Reviewers
 
@wumuzi520 SenlinZhu @Chong Li  @scv119 (Chen Shen) @jjyao (Jiajun Yao) @Yi Cheng
### Shepherd of the Proposal (should be a senior committer)


## Design and Architecture

### Brief idea
1. Introduce the concept of Label. Add the Labels attribute to Actor/Node. Labels = Map<String, String>.
2. After Actor are scheduled to a certain node, the Labels of Actor will be attached to the node resource(Named: LabelsResource). Node's Labels are naturally in the node resource.
3. Actor scheduling can choose Actor/NodeAffinitySchedulingStratgy. 
4. The actual principle of Actor/NodeAffinitySchedulingStratgy is. According to the label matching expression in the strategy, traverse and search for LabelsResource of all nodes. Pick out nodes that meet the expression requirements.

![LabelsAffinity](https://user-images.githubusercontent.com/11072802/203686866-385235b5-e08b-4aac-9c31-512621129bd4.png)

Scheduling Policy | Label Owner | Label select operator
-- | -- | --
ActorAffinity | Actor | in, not_in, exists, does_not_exist
NodeAffinity | Node | in, not_in, exists, does_not_exist

### API Design

**The apis of actors/nodes add labels**  

This interface is already very simple, so we will not set up multiple solutions for everyone to discuss.
```python
# Actor add labels.
actor_1 = Actor.options(labels={
    "location": "dc_1"
}).remote()


#  Node add static labels. 
ray start ... --labels={"location": "dc_1"}
# The api of the dynamic update node labels is similar to the current dynamic set_resource. It can be determined later.
```

**The apis of the actor-affinity/node-affinity scheduling.**  

**Option 1: Simplify through syntactic sugar**  

```python
actor_1 = Actor.options(
        scheduling_strategy=actor_affinity(label_in("location", ["dc_1"], false))
    ).remote()

actor_1 = Actor.options(
        scheduling_strategy=node_affinity(label_exist("location", false))
    ).remote()

actor_1 = Actor.options(
        scheduling_strategy=actor_affinity([
            label_in("location", ["dc_1"], false),
            label_exists("location", false)
        ])
    ).remote()

def actor_affinity(...):
    ...
    return ActorAffinitySchedulingStrategy(...)

def node_affinity(...):
    ...
    return NodeAffinitySchedulingStrategy(...)

def label_in(key, values, is_soft):
    ...
    return LabelMatchExpression(...)

def label_not_in(key, values, is_soft):
    ...
    return LabelMatchExpression(...)

def label_exists(key, is_soft):
    ...
    return LabelMatchExpression(...)

def label_does_not_exist(key, is_soft):
    ...
    return LabelMatchExpression(...)
```

**Option 2: another syntactic sugar** 

Personally, I think this Option is not as good as the above Option 1.  
The label_in(key, values, is_soft) form of option 1 is more understandable and better than the form of ("location", LabelMatchOperator.IN, ["dc_1"], false).
```python
actor_1 = Actor.options(
        scheduling_strategy=ActorAffinity([
            ("location", LabelMatchOperator.IN, ["dc_1"], false),
            ("location", LabelMatchOperator.Exist)).
    ).remote()
```

**Option 3: Java-like form**  

This form is similar to Java's syntax. The downside is that it's a bit complicated.
```python
SchedulingStrategyT = Union[None, str,
                            PlacementGroupSchedulingStrategy,
                            ActorAffinitySchedulingStrategy,
                            NodeAffinitySchedulingStrategy]

class ActorAffinitySchedulingStrategy:
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

actor_1 = Actor.options(scheduling_strategy=ActorAffinitySchedulingStrategy([
                LabelMatchExpression(
                    "location", LabelMatchOperator.IN, ["dc_1"], False)
            ])).remote()

actor_1 = Actor.options(scheduling_strategy=ActorAffinitySchedulingStrategy([
                LabelMatchExpression(
                    "location", LabelMatchOperator.IN, ["dc_1"], False),
                LabelMatchExpression(
                    "location", LabelMatchOperator.EXISTS, None, False)
            ])).remote()
```

**Option 4: Like sql**  

This solution is not recommended.  
This method needs to parse SQL, and the workload will be much larger.  
And users often write wrong sql when using it.
```python
# ActorAffinity use case
actor_1 = Actor.options(
        scheduling_strategy=ActorAffinity("location in [dc_1, dc_2]")
    ).remote()
```

### Example

* Affinity
  * Co-locate the actors in the same batch of nodes, like nodes in the same zones
* Anti-affinity
  * Spread the actors of a service across nodes and/or availability zones, e.g. to reduce correlated failures.

I will update the following API when the API plan is determined.  
Now describe the example with an api like java form.

**1. Spread Demo**

![spread demo](https://user-images.githubusercontent.com/11072802/207037933-8a9d9f1d-ee6e-472b-a877-669cef996db9.png)

```
@ray.remote
Class Cat:
    pass

cats = []
for i in range(4):
    cat = Actor.options(
            labels = {"type": "cat"},
            scheduling_strategy=ActorAffinitySchedulingStrategy([
                LabelMatchExpression(
                    "type", LabelMatchOperator.NOT_IN, ["cat"], False)
            ])).remote()
    cats.apend(cat)
```

**2. Co-locate Demo**

![co-locate demo](https://user-images.githubusercontent.com/11072802/207037951-df5bbf4a-442e-49e0-9561-39cfde45bf49.png)

```
@ray.remote
Class Dog:
    pass

dogs = []
# First schedule a dog to a random node.
dog_1 = Dog.options(labels={"type":"dog"}).remote()
dogs.apend(dog_1)

# Then schedule the remaining dogs to the same node as the first dog.
for i in range(3):
    dog = Actor.options(scheduling_strategy=ActorAffinitySchedulingStrategy([
                    LabelMatchExpression(
                        "type", LabelMatchOperator.IN, ["dog"], False)
                ])).remote()
    dogs.apend(dog)
```

**2. Collocate and spread combination demo**

![Collocate and spread combination demo](https://user-images.githubusercontent.com/11072802/207037895-125aab9d-d784-4a18-b777-1650c1d59226.png)

```
@ray.remote
Class Cat:
    pass

@ray.remote
Class Dog:
    pass

# First schedule cat to each node.
cats = []
for i in range(4):
    cat = Actor.options(
            labels = {
                "type": "cat",
                "id": "cat-" + str(i)
            },
            scheduling_strategy=ActorAffinitySchedulingStrategy([
                    LabelMatchExpression(
                        "type", LabelMatchOperator.NOT_IN, ["cat"], False)
            ])).remote()
    cats.apend(cat)

# Then each node schedules 3 dogs.
dogs = []
for i in range(4):
    node_dogs = []
    for i in range(3):
        dog = Actor.options(
                labels = {
                    "type": "dog",
                },
                scheduling_strategy=ActorAffinitySchedulingStrategy([
                        LabelMatchExpression(
                            "id", LabelMatchOperator.IN, ["cat-" + str(i)], False)
                ])).remote()
        node_dogs.apend(dog)
    dogs.apend(node_dogs)
```

### Note
1. Actor/NodeAffinity can be used together with the CustomResource mechanism.
These two mechanisms are completely non-conflicting.  
eg:
```
actor_1 = Actor.options(
                        resources={"4c8g": 1},
                        scheduling_strategy=ActorAffinitySchedulingStrategy([
                            LabelMatchExpression(
                                "location", LabelMatchOperator.IN, ["dc_1"], False)
                        ])).remote()
```
### Implementation plan

![LabelsAffinity](https://user-images.githubusercontent.com/11072802/203686851-894536d0-86fa-4fab-a5fe-2e656ac198e5.png)

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
  absl::flat_hash_map<string, absl::flat_hash_map<string, absl::flat_hash_map<string, string>>> labels;
}
```

2. Adapts where ResourcesData is constructed and used in the resource synchronization mechanism.  
a. NodeManager::HandleRequestResourceReport  
b. NodeManager::HandleUpdateResourceUsage 


3. Add Labels of actors/tasks to NodeResources during Actors/tasks scheduling


4. Scheduling optimization through Labels  
Take ActorAffinity's scheduling optimization scheme as an example：  
Now any node raylet has ActorLabels information for all nodes.  
when ActorAffinity schedules, if it traverses the Labels of all Actors of each node, the algorithm complexity is very large, and the performance will be poor.   
** Therefore, it is necessary to generate a full-cluster ActorLabels index table to improve scheduling performance. </b>

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
**Option 2: Use LabelAffinitySchedulingStrategy instead of Actor/Task/NodeAffinitySchedulingStrategy**  
Some people think that ActorAffinity is dispatched to the Node corresponding to the actor/Task with these labels.  
Why not assign both ActorLabels and TaskLabels to Node?   
Then the scheduling API only needs to use the LabelAffinitySchedulingStrategy set of APIs to instead of Actor/Task/NodeAffinitySchedulingStrategy.  

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

**Why didn't you choose this option?**   
1. Actor and Task are isolated by namespace. But Node has no namespace isolation. As a result, Actor/Task labels cannot be directly integrated with Node labels.
2. Separately divided into Actor/Task/NodeAffinity. In this way, the interface semantics will be more simple and concise. Avoid conceptual confusion.
3. LabelAffinity combines the labels of Actor/Task/Node at the same time. The final scheduling result may not be accurate. Users only want to use ActorAffinity. But other Tasks have the same labels with actor labels. This scene will make scheduling result not be accurate.

Advantages:
1. It is possible to realize the scene of affinity with Actor/Task/Node at the same time.  
Example:  
The user wants to affinity schedule to <b> some Actors and nodes in a special computer room. </b>   
However, according to the results of internal user research, most of the requirements are just to realize Actor/Task "collocate" scheduling or spread scheduling.  
So using a single ActorAffinity/NodeAffinity can already achieve practical effects.  

And the same effect can be achieved by combining the option 1 with custom resources


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


**Advantages:**
1. The interface for resource reporting and updating don't modify.

## Compatibility, Deprecation, and Migration Plan

## Test Plan and Acceptance Criteria
All APIs will be fully unit tested. All specifications in this documentation will be thoroughly tested at the unit-test level. The end-to-end flow will be tested within CI tests. Before the beta release, we will add large-scale testing to precisely understand scalability limitations and performance degradation in large clusters. 

## (Optional) Follow-on Work

### 1. Expression of "OR" semantics.
Later, if necessary, you can extend the semantics of "OR" by adding "is_or_semantics" to ActorAffinitySchedulingStrategy.
```
class ActorAffinitySchedulingStrategy:
    def __init__(self, match_expressions: List[LabelMatchExpression], is_or_semantics = false):
        self.match_expressions = match_expressions
        self.is_or_semantics = 
```

### 2. ObjectAffinitySchedulingStrategy
If the user has a request, you can consider adding the attributes of labels to objects. Then the strategy of ObjectAffinity can be launched。

### 3. TaskAffinitySchedulingStrategy
Because the resource synchronization mechanism of Label has been implemented above. Therefore, it is easy to create a TaskAffinity strategy for Task.

**Task add labels**
task_1 = Task.options(labels={
    "location": "dc_1"
}).remote()

**Add TaskAffinitySchedulingStategy**
ref = Task.options(
    scheduling_strategy = task_affinity(label_in("location", ["dc_1"], false)
).remote()

### 4. Use Affinity scheduling as another dimension scheduling strategy
The Actor/NodeAffinity strategy can be independent of the SchedulingStrategy as a second-dimensional scheduling strategy.  
Add a property of Affinity=Actor/NodeAffinity. eg:
```
actor_1 = Actor.options(
        scheduling_strategy="DEFAULT",
        affinity=actor_affinity(label_in("location", ["dc_1"], false)
    ).remote()
```