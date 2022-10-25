
## Summary
### General Motivation

Provides a set of lightweight node affinity and anti-affinity scheduling interfaces.
Replacing the heavy PG interface to implement collocate or spread actors.

* Affinity
  * Schedule an actor/task to a specified node or a range of nodes.
* Anti-affinity
  * Don't schedule an actor/task to a specified node or a range of nodes


### Should this change be within `ray` or outside?

Yes, this feature and actor affinity feature will be a complement to ray core's ability to flexibly schedule actors/tasks.

## Stewardship
### Required Reviewers
 
@wumuzi520 SenlinZhu @WangTaoTheTonic  @scv119 (Chen Shen) @jjyao (Jiajun Yao) 
### Shepherd of the Proposal (should be a senior committer)


## Design and Architecture
### current situation
The current NodeAffinity apis
```
actor = Actor.options(
    scheduling_strategy=NodeAffinitySchedulingStrategy(
        ray.NodeID.from_random().hex(), soft=True
    )
).remote()
```

Disadvantages now：  
1. No anti-affinity feature.
2. Only a single node can be specified, not a batch of nodes.
3. Now the nodeId is used as the identifier, so if the node id will change after the Failover occurs, the scheduling of Nodeaffinity will be inaccurate.

### Refactoring scheme

actor = Actor.options(
    scheduling_strategy=NodeAffinitySchedulingStrategy([
        NodeAffinityExpression(label_key, operator, [label_values], is_anti_affinity),
        NodeAffinityExpression(label_key, operator, [label_values], is_anti_affinity)
    ])
).remote()

The interface of NodeAffinity scheduling is similar to the label form of ActorAffinity。

Advantages:
1、Both support affinity and anti-affinity scheduling.
2、Both support a single or a batch of nodes.
3、Support user custom label
4、More flexible

### API Design
1、
ray start ... --node-name=xxx --node-label={"location":"dc-1"}

When Node init:
add node label:
{
    "node_id": node_id,
    "node_ip": node_ip,
    "node_name: node_name,
    "location": "dc-1",
}

#### Python API
Python API Design:  

Node affinity scheduling strategy API
```python
SchedulingStrategyT = Union[None, str,
                            PlacementGroupSchedulingStrategy,
                            NodeAffinitySchedulingStrategy,
                            ActorAffinitySchedulingStrategy]

class NodeAffinitySchedulingStrategy:
    def __init__(self, match_expressions: List[NodeAffinitySchedulingStrategy]):
        self.match_expressions = match_expressions

class NodeAffinitySchedulingStrategy:
    """An expression used to represent an node's affinity.
    Attributes:
        key: the key of label
        operator: IN、NOT_IN、EXISTS、DOES_NOT_EXIST,
                  if EXISTS、DOES_NOT_EXIST, values set []
        values: a list of label value
        soft: ...
    """
    def __init__(self, key: str, operator: NodeAffinityOperator,
                 values: List[str], soft: bool):
        self.key = key
        self.operator = operator
        self.values = values
        self.soft = soft

```



Step 2: Set actor affinity strategy.  
1. The target actor is expected to be scheduled with the actors whose label key is "location" and value in ["dc-1"].
```python
match_expressions = [
    NodeAffinityMatchExpression("node_id", ActorAffinityOperator.IN, ["xxx"], False)
]
node_affinity_strategy = NodeAffinitySchedulingStrategy(match_expressions)
node = Actor.options(scheduling_strategy = node_affinity_strategy).remote()
```


### Java API

Node affinity scheduling strategy API
```java
public class NodeAffinitySchedulingStrategy implements SchedulingStrategy {
  private NodeAffinitySchedulingStrategy(List<NodeAffinityMatchExpression> expressions) {
}

public class NodeAffinityMatchExpression {
  private String key;
  private NodeAffinityOperator operator;
  private List<String> values;
  private boolean isSoft;

  public static NodeAffinityMatchExpression in(String key, List<String> values, boolean isSoft) {
    return new NodeAffinityMatchExpression(key, NodeAffinityOperator.IN, values, isSoft);
  }

  public static NodeAffinityMatchExpression notIn(
      String key, List<String> values, boolean isSoft) {
    return new NodeAffinityMatchExpression(key, NodeAffinityOperator.NOT_IN, values, isSoft);
  }

  public static NodeAffinityMatchExpression exists(String key, boolean isSoft) {
    return new NodeAffinityMatchExpression(
        key, NodeAffinityOperator.EXISTS, new ArrayList<String>(), isSoft);
  }

  public static NodeAffinityMatchExpression doesNotExist(String key, boolean isSoft) {
    return new NodeAffinityMatchExpression(
        key, NodeAffinityOperator.DOES_NOT_EXIST, new ArrayList<String>(), isSoft);
  }

}

```

JAVA API example:  

Step 1: Set node affinity/anti-affinity strategy  
1. Scheduling into node of actor which the value of label key "location" is in {"dc-1", "dc-2"}
```java
List<String> nodes = new ArrayList<>();
nodes.add("xxxx");
nodes.add("aaaa");
NodeAffinitySchedulingStrategy schedulingStrategy =
    new NodeAffinitySchedulingStrategy.Builder()
        .addExpression(NodeAffinitySchedulingStrategy.in("node_id", nodes, false))
        .build();
ActorHandle<Counter> actor2 =
    Ray.actor(Counter::new, 1).setSchedulingStrategy(schedulingStrategy).remote();
```


### Implementation plan
.
#### GCS Scheduling Mode Implementation plan


## Test Plan and Acceptance Criteria
All APIs will be fully unit tested. All specifications in this documentation will be thoroughly tested at the unit-test level. The end-to-end flow will be tested within CI tests. Before the beta release, we will add large-scale testing to precisely understand scalability limitations and performance degradation in large clusters. 

## (Optional) Follow-on Work

