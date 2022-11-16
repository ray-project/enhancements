
## Summary
### General Motivation

Provides a set of lightweight actor affinity and anti-affinity scheduling interfaces.
Replacing the heavy PG interface to implement collocate or spread actors.

* Affinity
  * Co-locate the actors in the same node.
  * Co-locate the actors in the same batch of nodes, like nodes in the same zones
* Anti-affinity
  * Spread the actors of a service across nodes and/or availability zones, e.g. to reduce correlated failures.
  * Give a actor "exclusive" access to a node to guarantee resource isolation 
  * Spread the actors of different services that will affect each other on different nodes.


### Should this change be within `ray` or outside?

Yes, this will be a complement to ray core's ability to flexibly schedule actors/tasks.

## Stewardship
### Required Reviewers
 
@wumuzi520 SenlinZhu @Chong Li  @scv119 (Chen Shen) @jjyao (Jiajun Yao) 
### Shepherd of the Proposal (should be a senior committer)


## Design and Architecture

### Brief idea

1. Tag the Actor with key-value labels first
2. Then affinity or anti-affinity scheduling to the actors of the specified labels.

![ActorAffinityScheduling](https://user-images.githubusercontent.com/11072802/188054945-48d980ed-2973-46e7-bf46-d908ecf93b31.jpg)

Actor affinity/anti-affinity schedule API Design
1. Scheduling Strategy adds an ActorAffinitySchedulingStrategy. 
2. This strategy consists of several ActorAffinityMatchExpressions.
3. ActorAffinityMatchExpression has 4 match types which are IN/NOT_IN/EXISTS/DOES_NOT_EXIST

Use Case | ActorAffinityOperator
-- | --
Affinity | IN/EXISTS
Anti-Affinity | NOT_IN/DOES_NOT_EXIST

This is to learn the Affinity & Anti-Affinity features of K8s.

Advantage
1. Adding the Label mechanism makes affinity scheduling and anti-affinity scheduling particularly flexible. Actor scheduling of various topology types can be fully realized.
2. There can be many extensions to the actor's label mechanism, such as obtaining the actor based on the label.  



### API Design

#### Python API
Python API Design:  
Set key-value labels for actors
```python
class ActorClass:
    def options(self,
                .....
                labels: Dist[str, str] = None)
```

Actor affinity scheduling strategy API
```python
SchedulingStrategyT = Union[None, str,
                            PlacementGroupSchedulingStrategy,
                            NodeAffinitySchedulingStrategy,
                            ActorAffinitySchedulingStrategy]

class ActorAffinitySchedulingStrategy:
    def __init__(self, match_expressions: List[ActorAffinityMatchExpression]):
        self.match_expressions = match_expressions

class ActorAffinityMatchExpression:
    """An expression used to represent an actor's affinity.
    Attributes:
        key: the key of label
        operator: IN、NOT_IN、EXISTS、DOES_NOT_EXIST,
                  if EXISTS、DOES_NOT_EXIST, values set []
        values: a list of label value
        soft: ...
    """
    def __init__(self, key: str, operator: ActorAffinityOperator,
                 values: List[str], soft: bool):
        self.key = key
        self.operator = operator
        self.values = values
        self.soft = soft

```

Python API example:  
Step 1: Set the labels for this actor.   
Each label consists of a key and value of type string.
```python
actor_1 = Actor.options(labels={
    "location": "dc_1"
}).remote()
```

Step 2: Set actor affinity strategy.  
1. The target actor is expected to be scheduled with the actors whose label key is "location" and value in ["dc-1"].
```python
match_expressions = [
    ActorAffinityMatchExpression("location", ActorAffinityOperator.IN, ["dc_1"], False)
]
actor_affinity_strategy = ActorAffinitySchedulingStrategy(match_expressions)
actor = Actor.options(scheduling_strategy = actor_affinity_strategy).remote()
```

2. The target actor is not expected to be scheduled
with the actors whose label key is "location" and value in ["dc-1"].
```python
match_expressions = [
 ActorAffinityMatchExpression("location", ActorAffinityOperator.NOT_IN, ["dc_1"], False)
]
actor_affinity_strategy = ActorAffinitySchedulingStrategy(match_expressions)
actor = Actor.options(scheduling_strategy = actor_affinity_strategy).remote()
```

3. The target actor is expected to be scheduled with the actors whose label key exists "location".
```python
match_expressions = [
 ActorAffinityMatchExpression("location", ActorAffinityOperator.EXISTS, [], False)
]
actor_affinity_strategy = ActorAffinitySchedulingStrategy(match_expressions)
actor = Actor.options(scheduling_strategy = actor_affinity_strategy).remote()
```

4. The target actor is not expected to be scheduled with the actors whose label key exists "location".
```python
match_expressions = [
 ActorAffinityMatchExpression("location", ActorAffinityOperator.DOES_NOT_EXIST, [], False)
]
actor_affinity_strategy = ActorAffinitySchedulingStrategy(match_expressions)
actor = Actor.options(scheduling_strategy = actor_affinity_strategy).remote()
```

5. You can also set multiple expressions at the same time, and multiple expressions need to be satisfied when scheduling.
```python
match_expressions = [
 ActorAffinityMatchExpression("location", ActorAffinityOperator.DOES_NOT_EXIST, [], False),
 ActorAffinityMatchExpression("version", ActorAffinityOperator.EXISTS, [], False)
]
actor_affinity_strategy = ActorAffinitySchedulingStrategy(match_expressions)
actor = Actor.options(scheduling_strategy = actor_affinity_strategy).remote()
```

### Java API

Set the labels for this actor API
```java
  /**
   * Set a key-value label.
   *
   * @param key the key of label.
   * @param value the value of label.
   * @return self
   */
  public T setLabel(String key, String value) {
    builder.setLabel(key, value);
    return self();
  }

  /**
   * Set batch key-value labels.
   *
   * @param labels A map that collects multiple labels.
   * @return self
   */
  public T setLabels(Map<String, String> labels) {
    builder.setLabels(labels);
    return self();
  }
```

Actor affinity scheduling strategy API
```java
public class ActorAffinitySchedulingStrategy implements SchedulingStrategy {
  private ActorAffinitySchedulingStrategy(List<ActorAffinityMatchExpression> expressions) {
}

public class ActorAffinityMatchExpression {
  private String key;
  private ActorAffinityOperator operator;
  private List<String> values;
  private boolean isSoft;

  /**
   * Returns an affinity expression to indicate that the target actor is expected to be scheduled
   * with the actors whose label meets one of the composed key and values. eg:
   * ActorAffinityMatchExpression.in("location", new ArrayList<>() {{ add("dc-1");}}, false).
   *
   * @param key The key of label.
   * @param values A list of label values.
   * @param isSoft If true, the actor will be scheduled even there's no matched actor.
   * @return ActorAffinityMatchExpression.
   */
  public static ActorAffinityMatchExpression in(String key, List<String> values, boolean isSoft) {
    return new ActorAffinityMatchExpression(key, ActorAffinityOperator.IN, values, isSoft);
  }

  /**
   * Returns an affinity expression to indicate that the target actor is not expected to be
   * scheduled with the actors whose label meets one of the composed key and values. eg:
   * ActorAffinityMatchExpression.notIn( "location", new ArrayList<>() {{ add("dc-1");}}, false).
   *
   * @param key The key of label.
   * @param values A list of label values.
   * @param isSoft If true, the actor will be scheduled even there's no matched actor.
   * @return ActorAffinityMatchExpression.
   */
  public static ActorAffinityMatchExpression notIn(
      String key, List<String> values, boolean isSoft) {
    return new ActorAffinityMatchExpression(key, ActorAffinityOperator.NOT_IN, values, isSoft);
  }

  /**
   * Returns an affinity expression to indicate that the target actor is expected to be scheduled
   * with the actors whose labels exists the specified key. eg:
   * ActorAffinityMatchExpression.exists("location", false).
   *
   * @param key The key of label.
   * @param isSoft If true, the actor will be scheduled even there's no matched actor.
   * @return ActorAffinityMatchExpression.
   */
  public static ActorAffinityMatchExpression exists(String key, boolean isSoft) {
    return new ActorAffinityMatchExpression(
        key, ActorAffinityOperator.EXISTS, new ArrayList<String>(), isSoft);
  }

  /**
   * Returns an affinity expression to indicate that the target actor is not expected to be
   * scheduled with the actors whose labels exists the specified key. eg:
   * ActorAffinityMatchExpression.doesNotExist("location", false).
   *
   * @param key The key of label.
   * @param isSoft If true, the actor will be scheduled even there's no matched actor.
   * @return ActorAffinityMatchExpression.
   */
  public static ActorAffinityMatchExpression doesNotExist(String key, boolean isSoft) {
    return new ActorAffinityMatchExpression(
        key, ActorAffinityOperator.DOES_NOT_EXIST, new ArrayList<String>(), isSoft);
  }

}

```

JAVA API example:  
Step 1: Set labels for actors
```java
// Set a key-value label.
// This interface can be called multiple times. The value of the same key will be overwritten by the latest one.
ActorHandle<Counter> actor =
    Ray.actor(Counter::new, 1)
        .setLabel("location", "dc_1")
        .remote();

// Set batch key-value labels.
Map<String, String> labels =
    new HashMap<String, String>() {
        {
        put("location", "dc-1");
        put("version", "1.0.0");
        }
    };
ActorHandle<Counter> actor =
    Ray.actor(Counter::new, 1)
        .setLabels(labels)
        .remote();
```
Step 2: Set actor affinity/anti-affinity strategy  
1. Scheduling into node of actor which the value of label key "location" is in {"dc-1", "dc-2"}
```java
List<String> locationValues = new ArrayList<>();
locationValues.add("dc_1");
locationValues.add("dc_2");
ActorAffinitySchedulingStrategy schedulingStrategy =
    new ActorAffinitySchedulingStrategy.Builder()
        .addExpression(ActorAffinityMatchExpression.in("location", locationValues, false))
        .build();
ActorHandle<Counter> actor2 =
    Ray.actor(Counter::new, 1).setSchedulingStrategy(schedulingStrategy).remote();
```

2. Don't scheduling into node of actor which the value of label key "location" is in {"dc-1"}
```java
List<String> values = new ArrayList<>();
values.add("dc-1");
ActorAffinitySchedulingStrategy schedulingStrategyNotIn =
    new ActorAffinitySchedulingStrategy.Builder()
        .addExpression(ActorAffinityMatchExpression.notIn("location", values, false))
        .build();
ActorHandle<Counter> actor3 =
    Ray.actor(Counter::new, 1).setSchedulingStrategy(schedulingStrategyNotIn).remote();
```

3. Scheduling into node of actor which exists the label key "version"
```java
ActorAffinitySchedulingStrategy schedulingStrategyExists =
    new ActorAffinitySchedulingStrategy.Builder()
        .addExpression(ActorAffinityMatchExpression.exists("version", false))
        .build();
ActorHandle<Counter> actor4 =
    Ray.actor(Counter::new, 1).setSchedulingStrategy(schedulingStrategyExists).remote();
Assert.assertEquals(actor4.task(Counter::getValue).remote().get(10000), Integer.valueOf(1));
```

4. Don't scheduling into node of actor which exist the label key "version"
```java
ActorAffinitySchedulingStrategy schedulingStrategyDoesNotExist =
    new ActorAffinitySchedulingStrategy.Builder()
        .addExpression(ActorAffinityMatchExpression.doesNotExist("version", false))
        .build();
ActorHandle<Counter> actor5 =
    Ray.actor(Counter::new, 1).setSchedulingStrategy(schedulingStrategyDoesNotExist).remote();
```

5. You can also set multiple expressions at the same time, and multiple expressions need to be satisfied when scheduling.
```java
ActorAffinitySchedulingStrategy schedulingStrategy =
    new ActorAffinitySchedulingStrategy.Builder()
        .addExpression(ActorAffinityMatchExpression.doesNotExist("version", false))
        .addExpression(ActorAffinityMatchExpression.Exists("location", false))
        .build();
ActorHandle<Counter> actor6 =
    Ray.actor(Counter::new, 1).setSchedulingStrategy(schedulingStrategy).remote();
```

### Implementation plan
Now there are two modes of scheduling: GCS mode scheduling and raylet scheduling.  
It will be simpler to implement in GCS mode.
#### 1. GCS Scheduling Mode Implementation plan

1. Actor adds the Labels property. Stored in the GcsActor structure
2. Gcs Server add GcsLabelManager. Add labels->node information to GcsLabelManager after per actor completes scheduling.
3. Added ActorAffinityPolicy, whose scheduling logic is to select the main scheduling node based on each matching expression and the labels->node information of GcsLabelManager

The pseudo-code logic of the 4 matching operations is as follows:  
Affinity Operator: IN
```
feasible_nodes = FindFeasibleNodes()
for (expression : match_expressions) {
    feasible_nodes = feasible_nodes ∩ GcsLabelManager->GetNodesByKeyAndValue(expression.key, expression.values)
}
```

Affinity Operator: EXISTS
```
feasible_nodes = FindFeasibleNodes()
for (expression : match_expressions) {
    feasible_nodes = feasible_nodes ∩ GcsLabelManager->GetNodesByKey(expression.key)
}
```

Affinity Operator: NOT_IN
```
feasible_nodes = FindFeasibleNodes()
for (expression : match_expressions) {
    feasible_nodes = feasible_nodes - GcsLabelManager->GetNodesByKeyAndValue(expression.key, expression. values)
}
```

Affinity Operator: DOES_NOT_EXIST
```
feasible_nodes = FindFeasibleNodes()
for (expression : match_expressions) {
    feasible_nodes = feasible_nodes - GcsLabelManager->GetNodesByKey(expression.key)
}
```

GcsLabelManger Desion  

Add 4 public interfaces for GcsLabelManger :
```c++
// Add Actor's labels and node index relationship information.
void AddActorLabels(const std::shared_ptr<GcsActor> &actor);

// Remove Actor's labels and node index relationship information.
void RemoveActorLabels(const std::shared_ptr<GcsActor> &actor);

// Get the node where the actors those keys and values matched. 
absl::flat_hash_set<NodeID> GetNodesByKeyAndValue(const std::string &ray_namespace, const std::string &key, const absl::flat_hash_set<std::string> &values) const;

// Get the node where the actors which exist this key.
absl::flat_hash_set<NodeID> GetNodesByKey(const std::string &ray_namespace, 
                                          const std::string &key) const;
```

Main data structure :
```
Map<label_key, Map<lable_value, Set<node_id>>> label_to_nodes_
Map<node_id, Set<GcsActor>> node_to_actors_
```
#### 2. Raylet Scheduling Mode Implementation plan
The implementation of Raylet scheduling mode is same as GCS scheduling above.  
Mainly, one more Labels information needs to be synchronized to all Raylet nodes

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

  // Map<key, Map<value, reference_count>> Actors scheduled to this node and actor labels information
  repeat Map<string, Map<string, int>> actor_labels = 15
  // Whether the actors of this node is changed.
  bool actor_labels_changed = 16,
}


NodeResources {
  ResourceRequest total;
  ResourceRequest available;
  /// Only used by light resource report.
  ResourceRequest load;
  /// Resources owned by normal tasks.
  ResourceRequest normal_task_resources
  /// Actors scheduled to this node and actor labels information
  absl::flat_hash_map<string, absl::flat_hash_map<string, int>> actor_labels;
}
```

2. Adapts where ResourcesData is constructed and used in the resource synchronization mechanism.  
a. NodeManager::HandleRequestResourceReport  
b. NodeManager::HandleUpdateResourceUsage 


3. Add ActorLabels information to NodeResources during Actor scheduling

a. When the Raylet is successfully scheduled, the ActorLabels information is added to the remote node scheduled in the ClusterResoucesManager.  
```
void ClusterTaskManager::ScheduleAndDispatchTasks() {
    auto scheduling_node_id = cluster_resource_scheduler_->GetBestSchedulableNode(
    ScheduleOnNode(node_id, work);
    	cluster_resource_scheduler_->AllocateRemoteTaskResources(node_id, resources)
        cluster_resource_scheduler_->GetClusterResourceManager().AddActorLabels(node_id, actor);
```
b. Add ActorLabels information to LocalResourcesManager when Actor is dispatched to Worker.
```
LocalTaskManager::DispatchScheduledTasksToWorkers()
   cluster_resource_scheduler_->GetLocalResourceManager().AllocateLocalTaskResources
   cluster_resource_scheduler_->GetLocalResourceManager().AddActorLabels(actor)
   worker_pool_.PopWorker()
```

c. When the Actor is destroyed, the ActorLabels information of the LocalResourcesManager is also deleted.
```
NodeManager::HandleReturnWorker
  local_task_manager_->ReleaseWorkerResources(worker);
    local_resource_manager_->RemoveActorLabels(actor_id);
```

Actor scheduling flowchart：
![Actor scheduling flowchart](https://user-images.githubusercontent.com/11072802/202128385-f72609c5-308d-4210-84ff-bf3ba6df381c.png)

Node Resources synchronization mechanism:
![Node Resources synchronization mechanism](https://user-images.githubusercontent.com/11072802/202128406-b4745e6e-3565-41a2-bfe3-78843379bf09.png)

4. Scheduling optimization through ActorLabels  
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

<b> VS.  putting Labels into the custom resource solution  <b>  
<b>Advantages:<b>
1. Compared with the scheme of putting Labels in the custom resource. This scheme can also reuse the resource synchronization mechanism. Then it won't destroy the concept of coustrom resouce.
2. The Label index table of all nodes can be constructed from the ActorLabels information of each node. If you use Custom Resource, you can't build.
3. If the Custom Resouces scheme is used, the accuracy of custom resouces scheduling will be affected. The current scheme is completely independent of existing scheduling policies and resources and will not affect them. The code is also more concise.


<b>DisAdvantages:<b>
1. The interface for resource reporting and updating needs to be adapted to ActorLabels in ResouceData.

<b>Issue<b>
1. Because there must be a delay in resource synchronization under raylet scheduling. So if actor affinity is Soft semantics, there will be inaccurate scheduling.


### Failures and Special Scenarios
#### 1、If the Match Expression Cannot be satisfied
If the matching expression cannot be satisfied, The actor will be add to the pending actor queue. Util the matching expression all be statisfied。

Example:
If the next actor want to co-locate with the previous actor but the previous actor don't complete schedule. The next actor will be add to pending actor queue. It will be schedule complete After the previous actor complete schedule.

#### 2、Failures Scenarios
If actor B collocates with actor A and The actor A & B both complete schedule to Node 1.
1、If actor A failed and restarted to Node 2, the actor B will not be affected, it still working in Node 1.  
2、If actor A failed and restarted to Node 2, Then the actor B also failed, It will be scheduled to Node 2 and collocates with actor A.

### how other system achieve the same goal?
1、K8s
This solution is to learn the PodAffinity/NodeAffinity features of K8s。
https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity

### what's the alternative to achieve the same goal?
1、Ray 
Now ray placement group can achieve the same goal. But PG is too heavy and complicated to be user friendly
## Compatibility, Deprecation, and Migration Plan

## Test Plan and Acceptance Criteria
All APIs will be fully unit tested. All specifications in this documentation will be thoroughly tested at the unit-test level. The end-to-end flow will be tested within CI tests. Before the beta release, we will add large-scale testing to precisely understand scalability limitations and performance degradation in large clusters. 

## (Optional) Follow-on Work

### Expression of "OR" semantics.
Later, if necessary, you can extend the semantics of "OR" by adding "is_or_semantics" to ActorAffinitySchedulingStrategy.
```
class ActorAffinitySchedulingStrategy:
    def __init__(self, match_expressions: List[ActorAffinityMatchExpression], is_or_semantics = false):
        self.match_expressions = match_expressions
        self.is_or_semantics = 
```