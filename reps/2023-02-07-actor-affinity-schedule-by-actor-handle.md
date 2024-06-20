
## Summary
### General Motivation

Provides a set of lightweight actor affinity and anti-affinity scheduling interfaces.
Replacing the heavy PG interface to implement collocate or spread actors.

What users complain about now?
1. Using PG to achieve affinity and anti-affinity between actors is too cumbersome. It is necessary to create PG first and estimate the required resources.
2. PG cannot dynamically expand and contract, and bundle resources cannot be dynamically changed. PG cannot be used in scenes where the number of Actors is uncertain.
3. Now the PG creation process is serial and complicated, resulting in slow performance.

So now the scheduling strategy of ActorAffinity is introduced. It can directly meet the needs of affinity and anti-affinity between Actors, and it is much more convenient to use and faster in performance.

### Should this change be within `ray` or outside?

Yes, this will be a complement to ray core's ability to flexibly schedule actors/tasks.

## Stewardship
### Required Reviewers
@ericl @stephanie-wang @wumuzi520 SenlinZhu @Chong Li  @scv119 (Chen Shen) @Sang Cho @jjyao (Jiajun Yao) @Yi Cheng
### Shepherd of the Proposal (should be a senior committer)


## Design and Architecture
### API Design
```python
actor_0 = Actor.remote()

# Affinity to actor_0.
actor_1 = Actor.options(
        scheduling_strategy=actor_affinity(affinity(actor_0, false))
    ).remote()

# Affinity to actor_0 or actor_1.
actor_2 = Actor.options(
        scheduling_strategy=actor_affinity(affinity([actor_0, actor_1], false))
    ).remote()

# Anti-affinity to actor_0.
actor_3 = Actor.options(
        scheduling_strategy=actor_affinity(anti_affinity(actor_0, false))
    ).remote()

# Anti-affinity to both actor_0 and actor_1.
actor_4 = Actor.options(
        scheduling_strategy=actor_affinity([
            anti_affinity(actor_0, false),
            anti_affinity(actor_1, false)
            ])
    ).remote()



def actor_affinity(expressions: LabelMatchExpression[]):
    ...
    return ActorAffinitySchedulingStrategy(...)

def affinity(actor_handles, is_soft):
    ...
    return LabelMatchExpression(...)

def anti_affinity(actor_handles, is_soft):
    ...
    return LabelMatchExpression(...)
```


### Implementation plan
**Now the api provides a way to pass actor_handles. But the specific internal implementation is still implemented through actor labels. It is convenient to extend the ActorAffinity interface based on the actor labels.**

Now the core logic of gcs scheduling and raylet scheduling have been merged and unified. Therefore, in the implementation of ActorAffinity, both Gcs scheduling and raylet scheduling reuse the same code, but in the raylet scheduling mode, there is an additional synchronization mechanism of actor labels.

1. The GcsActor structure add labels property.  

This Actor Labels is only available for internal implementation, and the user cannot perceive it.
```
# There is only one label in Actor Labels which is {"actor_id": actor_id}.
class GcsActor {
    actor_id;
    labels = {
        "actor_id": actor_id
    };
}
```

2. When an actor is dispatched to a node, add the actor's labels and node_id information to the node resources and the ClusterActorLabelManager.

> 1. {Actor labels -> node} infos add to Node Resources is for synchronizing Labels information in Raylet scheduling mode. This will be mentioned later in a separate chapter.
> 2. The ClusterActorLabelManager is actually an index table of actor_id(actor_labels) and node_id. {Actor labels -> node} infos add to ClusterActorLabelManager is for directly obtain the corresponding node_id through actor_id (actor_labels) when scheduling. It is used to improve performance.

```
class ClusterActorLabelManager {
 public:
  absl::flat_hash_set<NodeID> GetNodesByKeyAndValue(const std::string &key, const absl::flat_hash_set<std::string> &values) const;
  absl::flat_hash_set<NodeID> GetNodesByKey(const std::string &key) const;
  void AddActorLabels(const std::shared_ptr<GcsActor> &actor);
  void RemoveActorLabels(const std::shared_ptr<GcsActor> &actor);
 private:
 <namespace, <label_key, <lable_value, <node_id, ref_count>>>> labels_to_nodes_;
}

# Note: This part of code ActorAffinity and NodeAffinity will be reused.
```

3. Added ActorAffinityPolicy, whose scheduling logic is to select the main scheduling node based on each matching expression and the actor_id -> node information of ClusterActorLabelManager.

The detail logic is as follows:
1. actor_affinity(affinity(actor_0, false)
```
actor_0 = Actor.remote()
# Affinity to actor_0.
actor_1 = Actor.options(
            scheduling_strategy=actor_affinity(affinity(actor_0, false))
          ).remote()
```

```
# The above API is actually a syntactic sugar, which will actually be transformed into:
affinity(actor_0, false) 
-> 
LabelMatchExpression(key="actor_id", match_operator="IN", values=["actor_0"], is_soft=false)
```

```
actor_affinity(affinity(actor_0, false)) 
-> 
ActorAffinitySchedulingStrategy([
    LabelMatchExpression(key="actor_id", match_operator="IN", values=["actor_0"], is_soft=false)
])
```

```
actor_1 = Actor.options(
            scheduling_strategy=actor_affinity(affinity(actor_0, false))
          ).remote()
->
actor_1 = Actor.options(
            scheduling_strategy=ActorAffinitySchedulingStrategy([
                LabelMatchExpression(key="actor_id", match_operator="IN", values=["actor_0"], is_soft=false),
            ])
          ).remote()

# The final scheduling logic is:
feasible_nodes = FindFeasibleNodes()
feasible_nodes = feasible_nodes ∩ ClusterActorLabelManager->GetNodesByKeyAndValue("actor_id", ["actor_0"])
```

2. actor_affinity(anti_affinity(actor_0, false)
```
actor_affinity(anti_affinity(actor_0, false) 
->
ActorAffinitySchedulingStrategy([
    LabelMatchExpression(key="actor_id", match_operator="NOT_IN", values=["actor_0"], is_soft=false),
])

# The final scheduling logic is:
feasible_nodes = FindFeasibleNodes()
feasible_nodes = feasible_nodes - ClusterActorLabelManager->GetNodesByKeyAndValue("actor_id", ["actor_0"])
```

3.actor_affinity([
            anti_affinity(actor_0, false),
            anti_affinity(actor_1, false)
            ])

```
actor_affinity([
            anti_affinity(actor_0, false),
            anti_affinity(actor_1, false)
            ])
->
ActorAffinitySchedulingStrategy([
    LabelMatchExpression(key="actor_id", match_operator="NOT_IN", values=["actor_0"], is_soft=false),
    LabelMatchExpression(key="actor_id", match_operator="NOT_IN", values=["actor_1"], is_soft=false)
])

# The final scheduling logic is:
feasible_nodes = FindFeasibleNodes()
feasible_nodes = feasible_nodes - ClusterActorLabelManager->GetNodesByKeyAndValue("actor_id", ["actor_0"])
feasible_nodes = feasible_nodes - ClusterActorLabelManager->GetNodesByKeyAndValue("actor_id", ["actor_1"])
```

#### 2. ActorLabels information synchronization in Raylet scheduling mode
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

  // Map<label_type, Map<namespace, Map<label_key, Map<label_value, ref_count>>>> Actors/Tasks/Nodes labels information
  repeat Map<string, Map<string, Map<string, Map<string, int>>> labels = 15
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
  /// Map<label_type, Map<namespace, Map<label_key, Map<label_value, ref_count>>>> Actors/Tasks/Nodes labels information
  absl::flat_hash_map<string, absl::flat_hash_map<string, absl::flat_hash_map<string, absl::flat_hash_map<string, int>>>> labels;
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
![Node Resources synchronization mechanism](https://user-images.githubusercontent.com/11072802/203783157-fad67f25-b046-49ac-b201-b54942073823.png)


**Issue**
1. Because there must be a delay in resource synchronization under raylet scheduling. So if actor affinity is Soft semantics, there will be inaccurate scheduling.

    a. If the user selects the Soft strategy. That means that the user can accept the fact that the scheduling has a certain probability of being inaccurate.

    b. Most users schedule a batch of actors on the same node. In this case we can do exactly the right thing.


### Failures and Special Scenarios
#### 1、If the Match Expression Cannot be satisfied
If the matching expression cannot be satisfied, The actor will be add to the pending actor queue. Util the matching expression all be statisfied。

1. resources are enough, but affinity cannot be satisfied? -> Hang & report schedule failed event and detail unstaisfed reason to exposed to users

2. affinity can be satisfied, but resources are not enough? -> Hang & report schedule failed event and detail unstaisfed reason to exposed to users

3. both affinity and resources are not satisfied? -> Hang & report schedule failed event and detail unstaisfed reason to exposed to users


Example:
If the next actor want to co-locate with the previous actor but the previous actor don't complete schedule. The next actor will be add to pending actor queue. It will be schedule complete After the previous actor complete schedule.


#### 2、Failures Scenarios
If actor B collocates with actor A and The actor A & B both complete schedule to Node 1.
1. If actor A failed and restarted to Node 2, the actor B will not be affected, it still working in Node 1.  
2. If actor A failed and restarted to Node 2, Then the actor B also failed, It will be scheduled to Node 2 and collocates with actor A.

### AutoScaler adaptation
Now AutoScaler is just restarting, so the main consideration is to adapt to the new version of AutoScaler.

AutoScaler is divided into two parts.
1. Interaction API between AutoScaler and gcs.
I think I will adapt it according to the original method at that time, Mainly adding new fields.

2. AutoScaler policy with node affinity (which decide what node needs to add to the cluster)  
    1. affinity to a special actor  
        Because the user specifies the affinity to the node of an actor. So this scene does not need to add nodes.
    2. anti-affinity to a special actor  
        Because it is anti-affinity, you can choose to add nodes for capacity expansion in this case.
    3. soft affinity to special actor  
        Because it is a soft strategy, you can choose to add new nodes when the existing nodes cannot meet the requirements.

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
**Why is it implemented with actor labels instead of directly synchronizing actor_id and node_id information？**  
1. There is not much difference in the amount of code changes for these two implementations. Both resource synchronization and index tables for actor_id and node_id are required.
2. Using ActorLabels can easily extend the ActorAffinity strategy based on labels
3. The way of using Actorlabels can reuse code with NodeAffinity.

## Compatibility, Deprecation, and Migration Plan

## Test Plan and Acceptance Criteria
All APIs will be fully unit tested. All specifications in this documentation will be thoroughly tested at the unit-test level. The end-to-end flow will be tested within CI tests. Before the beta release, we will add large-scale testing to precisely understand scalability limitations and performance degradation in large clusters. 

## (Optional) Follow-on Work

### 1. Extend the ActorAffinity strategy based on labels
ActorAffinity usage scenarios based only on actor_handle are relatively simple. In FailOver scenarios, there will be problems. Therefore, being able to enter ActorAffinity based on Labels in the future will greatly enrich the support scenarios.

```python
# Actor add labels.
actor_1 = Actor.options(labels={
    "location": "dc_1"
}).remote()

actor_1 = Actor.options(
        scheduling_strategy=actor_affinity(label_in("location", ["dc_1"], false))
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