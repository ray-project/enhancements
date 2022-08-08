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

We implement
#### Options to implement object HA with checkpoint

We implement object HA based on the checkpoint, so we can walk around **Problem 3: Loss of All Copies**

##### Option 1: Ownerless objects

Every checkpointed object will be ownerless. When `ray.get` on a checkpointed object and the object data is not local, data will be loaded from checkpoint and stored in local object store as a secondary copy.

In this case, We don't need metadata anymore, so we don't need to manage it anymore, and **problems 1 and 2** don't exist.

- Pros
  - The logic/protocol should be relatively straightforward.
- Cons
  - No pulling/pushing object data between nodes.
    - High IO pressure on external storage.
    - External storage bottleneck.
    - Bad loading performance.
  - Existing code is purely ownership-based. Need to add many if-elses to support ownerless objects. (e.g. location pubsub. GC.)
    - High dev cost.
    - Maintenance burden.
  - Some features aren't easy to support. e.g. locality-aware scheduling.
    - Missing features.

##### Option 2: Divided owners on owner failure

Once the owner of a checkpointed object is dead, subsequent access to the object on a worker will make the worker the new owner of the object. The metadata about this object in the reference table of the worker will be rewritten. If multiple workers hold the same object ref and want to access it after owner failure, each worker will become the new owner of the object, independently. i.e. multiple owners after owner failure.

If both worker A and B are the new owners of an object and both pass object refs to worker C, worker C only records the owner of the object once. e.g. If worker C receives the object ref from worker A first, then worker C treats worker A as the owner of the object.

**Problem 2** will cause the owner to release the reference of the object in advance. In the current case, it can be handled in the same way as the owner fails.

- Pros
  - The ownership protocol is still distributed. Critical updates to the owner reference table are still in-process.
    - Good performance on metadata updates.
- Cons
  - Owner collision. e.g. Raylet also stores and uses owner addresses and communicates with owners. How do we update the owner info in Raylet, especially if two workers on the same node claim the new owner of an object?
    - High dev cost to sync new owner info to Raylet.
    - High dev cost to maintain N owners of an object in Raylet.
  - Primary copy collision. Imaging multiple workers on the same node call `ray.get` on an object.
    - If we allow only one copy loaded into the local object store, we need to update the GC strategy to only unpin the primary copy if all owners say the object is out-of-scope.
      - High dev cost.
    - If we store multiple copies (i.e. one copy per owner), we need to update object manager and plasma store by changing the object key from object ID to object ID + owner address.
      - High dev cost.
      - High memory pressure to object store.
      - High IO pressure on external storage.
      - External storage bottleneck.
      - Bad loading performance.
  - No pulling/pushing object data between nodes.
    - High IO pressure on external storage.
    - External storage bottleneck.
    - Bad loading performance.
  - Corner case handling such as RPC failures.
    - Potentially high dev cost.

##### Option 3: Global owner(s)

We use highly available processes as global owners of checkpointed objects. Such highly available processes can be GCS or a group of named actors with `max_restarts=-1`. We reuse the existing ownership assignment RPCs to assign a checkpointed object to a global owner and encode the immutable info (an `owner_is_gcs` flag or the actor name) about the global owner into the owner address. The process to get an RPC client to the owner needs to be updated to be able to return a working RPC client to the up-to-date IP:port of the owner.

Note that we don't need to restore the reference table in global owners by pulling info from the cluster because objects are already checkpointed. Checkpoint info is stored in the reference table and it will be encoded when serializing an object ref, hence checkpoint info is recoverable. If borrowers detected owner failure, they will try to reconnect to the owner and the recovered owner will recover the reference count and borrower list via these new RPC connections.

- Pros
  - No major protocol changes compared to the existing ownership assignment protocol.
    - Low dev cost.
  - No owner address updates because the `owner_is_gcs` flag or the actor name is encoded in it.
    - Low dev cost.
- Cons
  - Centralized/semi-centralized architecture.
    - Potentially bottleneck.
    - Bad performance.
  - Corner case handling such as RPC failures.
    - Potentially high dev cost.

We prefer named actors rather than GCS as global owners.

- The number of global owners is configurable, hence scalable.
- No need to embed (part of) core worker code into GCS.
- No increased complexity for GCS.

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

- **Prototype test**
