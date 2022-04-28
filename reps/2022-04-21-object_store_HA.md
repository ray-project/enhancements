## Summary - Object Store High Availability

### General Motivation

Ray is a general-purpose and powerful computing framework. With the Object Store, it can be easily extended into a data service to provide data for various distributed tasks. Ray uses decentralized object ownership to avoid centralized bottlenecks in managing an object’s metadata (mainly its reference count and object directory entries), but it difficult to handle failover.

For now, users can only rely on lineage to recover the unavailable object. But lineage has many restriction:
- Can not recover the object which put in object store via `ray.put`.
- Can not recover the object returned by actor task.
- Require task is idempotent.

#### Goal

1. Objects can be specified for high availability mode, other objects are the same as before.
2. Any high availability objects should still be accessible if we encounter a single-node failure.

### Should this change be within `ray` or outside?

Changes are within Ray core.

## Stewardship

### Required Reviewers

@stephanie-wang, @ericl, @scv119, @kfstorm, @raulchen

### Shepherd of the Proposal (should be a senior committer)

@ericl, @raulchen, @stephanie-wang

## Design and Architecture

### Problem statement

#### Problem 1: Object Owner Failure

The owner of an object stores the metadata of the object, such as reference count and locations of the object. If the owner dies, other workers which hold the object ref cannot access the data of the object anymore because all copies of the object will be deleted from the Object Store.

#### Problem 2: Object Borrower Failure

In the chained object borrowing case, the owner of an object is not aware of the indirect borrowers. If all direct borrower fails, the owner will consider the object out-of-scope and GC the object. Accessing the object on indirect borrowers will fail with an `ObjectLostError`.

more details: [issues 18456](https://github.com/ray-project/ray/issues/18456)

#### Problem 3: Loss of All Copies

Data of objects stored in the plasma store. For now, the plasma store is a thread of the raylet process, failure of the raylet process will lose data which store in plasma. Some objects which only one copy in that failed plasma store,  will be unavailable.

### Proposed Design

#### How to solve the object owner failure problem?

**Global Owner**: A group of special Ray actors with `max_restarts=-1` will be created to own all objects of a job created by `ray.put` and the returned object (either from normal-task or actor-task). When one of the special actors fails, the restarted instance will rebuild the reference table based on information provided by Raylets and other non-special workers.

These actors will be distributed onto different nodes with best effort. A job-level configuration will be added (disabled by default) to enable this feature and set the number of the global owners.

Just need to consider how to restore data on B after A restarts, as well as rebuild various RPC links and recover subscriptions, such as `WaitForRefRemoved` and `WaitForObjectFree`.


##### Option 1: Syncing reference table of owners to a highly available external storage.

We keep the in-memory reference table of owners in sync with a high-availability external storage. When an owner restarts from failure, it can restore the reference table from the external storage and recreate the RPC connections with borrowers and Raylets.

**Pros**:

- Less intrusive to the design of object store.

**Cons**:

- Deployment is more complicated.
- The modification of the reference table is a high-frequency operation. Syncing with an external storage may hurt performance.
- Potentially some tricky consistency issues when an owner fails.

##### Option 2: Rebuild failed global owner via the information in the whole cluster

There are two options, the active way and passive way, to rebuild the reference table and the RPC connections when a global owner restarts.

**The Active Way**:

The global owner actively collect information about the alive objects it owns to rebuild the reference table. Here are the details steps of global owner `G` rebuild:

1. `G` begins to rebuild the reference table, and sets the rebuilding status to `REBUILDING`.
2. `G` sends RPCs to all Raylets and ask every Raylet to traverse all local workers and reply with the information of all objects owned by `G`. `G` then uses the information to rebuild the reference table and re-establish RPC connections.
3. `G` sets the rebuilding state to `READY`.

**Note**:

- RPCs sent by other workers to `G` should be retried until the state of `G's` Actor becomes `Alive`.
- The reply callback of some RPCs will not be invoked until the rebuilding state of `G` is set to `READY`.

**The Passive Way**:

Every Raylet maintains an RPC connection to every global owner to watch the status of the global owners. When the global owner `G` dies and the job is not finished yet. Each raylet will collect and merge the reference table of all workers on this node, and report to `G`. Details:

1. Raylet will find out that `G` is dead through RPC disconnection.
2. When Raylet knows that `G` is restarted, Raylet sends the information below to `G`:
    - References of all objects which are owned by `G`. (Collected by traversing all local workers.)
    - Objects which are owned by `G` and are local (the data is in the local Object Store).
    - The spill URLs of locally spilled objects which are owned by `G`.
3. `G` will reconnect to the Raylet and the workers on it after receiving the above collected information from this Raylet.
4. `G` sets the state to `READY` after finished rebuilding reference table.
5. The reply callback of some RPCs (`WaitForObjectFree`) will not be called until the rebuilding state of `G` is set to `READY`.

**Note**:

- RPCs sent by other workers to `G` should be retried until the state of `G's` Actor becomes `Alive`.
- The reply callback of some RPCs will not be invoked until the rebuilding state of `G` is set to `READY`.

We prefer __Option.2__, and due to the difficulty of implementation, we are more trend to choose the __Active__ way to restore `G`, here are some cons of the __Passive__ way:

- Raylets need to subscribe to the actor state notifications in order to know the new RPC address of a restarted actor and reconnect to it. To implement this, we need to add something similar to "actor manager" in core worker to Raylet. This is an additional coding effort.
- Raylets pushing collected information to the restarted actor v.s. the actor pulling information from Raylets is just like push-based v.s. pull-based resource reporting. This is what we've already discussed and we've concluded that pull-based resource reporting is easier to maintain and test. I believe this conclusion also applies to object reference and location reporting.



**Pros**:

- No need to rely on an external storage.
- Performance will not be affected.

**Cons**:

- **Active**: The rebuilding process may take a long time.
- **Passive**: When a global owner is waiting for collected information from Raylets, the global owner needs to handle timeouts, which can be difficult.
- Relatively complicated.

#### How to solve the object borrower failure problem?

##### Star Topology

We use star topology instead of tree topology when the number of global owners is greater than zero to make sure the owner directly lends an object to other workers. As the illustration shows:

![image_1_star_topology](https://user-images.githubusercontent.com/11995469/165585288-c6fc4ba4-efd6-42b5-935b-ea5979d0e735.png)

1. `Worker C` borrows `Object A` from `Worker B` as before, and adds `Worker C` to `Worker B's` borrowers list.
2. Send a sync RPC to `G`, and borrows `Object A` from `G` when `object A` deserializes in `Worker C`.
3. Send an async RPC to delete `Worker C` from borrowers on `Worker B`.

#### How to solve the loss of all copies problem?

##### Multiple Primary Copies

We make sure there are multiple (configurable) primary copies of the same object. We can create these additional copies asynchronously to reduce performance penalty, and creating another copy once the object is sealed.

#### API:

``` python
# Set the number of global owner (default is zero) and the number of HA object's primary copies (default is zero).
ray.init(
    job_config=ray.job_config.JobConfig(
        num_global_owners=16,
        num_primary_copies=3,
    )
)

# put a HA object. the default value of `enable_ha` is False.
ray.put(value, enable_ha=True)

# normal task: returns HA object.
# the default value of `enable_ha_for_return_objects` is False.
@ray.remote(enable_ha_for_return_objects=True)
def fun(*args, **kwargs):
    ...

# actor task: returns HA object.
# the default value of `enable_ha_for_return_objects` is False.
@ray.remote(enable_ha_for_return_objects=True)
class Actor:
    def func(self, *args, **kwargs):
        ...

```


## Compatibility, Deprecation, and Migration Plan

All these features in this REP are optional. The default behavior is the exactly the same as before. Users need to explicitly configure new options to enable these features.

## Test Plan and Acceptance Criteria

We plan to use a Ray job to test the HA feature of the Object Store.

1. In a multi-node cluster, each node runs two types of actors: producer and consumer.
    - Each **producer** actor produces data and stores object refs at local. Adds or deletes objects according to a certain strategy for testing object GC.
    - Each **consumer** actor gets an actor handle of a producer actor via the actor name and borrow objects from the producer actor randomly through `ray.get`.
2. Adjust data scale according to parameters:
    - The size of an object.
    - The number of objects.
    - The capacity of the Object Store.

Acceptance criteria:

1. Performance degradation is acceptable when no process or node failures happen.
2. When a single worker process or Raylet fails, the test job can finish eventually.

## (Optional) Follow-on Work

- **Global Owner**: Automatically adjust the number of global owner according to the number of objects in the ray cluster.
- **Star Topology**: Change the synchronous RPC sent to `G` when deserializing on `Worker C` to asynchronous.
- **Multiple Primary Copies**: Recreate new primary copies upon loss of any primary copy to meet the required number of primary copies.
