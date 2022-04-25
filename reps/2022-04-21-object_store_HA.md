## Summary - Object Store High Availability

### General Motivation

Ray is a general-purpose and powerful computing framework. With the Object Store, it can be easily extended into a data service to provide data for various machine learning tasks. Here are two typical scenarios.

#### Scenarios 1: Data Provider for Deep Learning Offline Training

We can implement a data provider on top of Ray with the following functions:

1. Reading data from various data sources and storing them in the Object Store, which can be read by workers.
2. Simple data processing capability, such as data augmentation, data distillation, .etc.

#### Scenarios 2: Mars on Ray

[Mars](https://github.com/mars-project/mars) is a tensor-based, unified framework for large-scale data computation which scales numpy, pandas, scikit-learn and many other libraries.

Mars on Ray uses Ray actors as execution backend. Mars leverages the Object Store for data exchanges between Mars workers.

#### Why do we need high availability for the Object Store?

There are two requirements from the above scenarios:

1. **Stability**: As a long-running service, workers need to be automatically restored after failures. Currently, a worker failure may result in object loss due to the worker is the owner of these objects.
2. **Auto-Scaling**: The cluster needs to be scaled down when the workload is reduced. Currently, dropping a node may result in object loss if the only copy of an object is on this node.

**TODO: Node failure is not mentioned in stability.**

**TODO: Why these issues exist in these scenarios but not the others (e.g. Ray dataset)? Need explanation.**

### Goal

All objects should still be accessible if we encounter a single-node failure.

### Should this change be within `ray` or outside?

Changes are within Ray core.

## Stewardship

### Required Reviewers

@stephanie-wang, @ericl, @scv119

### Shepherd of the Proposal (should be a senior committer)

@ericl

## Design and Architecture

### Problem statement

**TODO: Is it better to have a separate section or separate document for the detailed design and let this REP mainly focus on motivation and high-level design?**

#### Problem 1: Object Owner Failure

The owner of an object stores the metadata of the object, such as reference count and locations of the object. If the owner dies, other workers which hold the object ref cannot access the data of the object anymore because all copies of the object will be deleted from the Object Store.

For example, if the owner of the object `O_A` is on node `N_A`, the object `O_A` will be out-of-scope when node A dies. All copies of the object will be deleted, and any worker which wants to access `O_A` will fail with an `ObjectLostError`.

**TODO: Naming refine. `O_A` -> `Object A`, `W_A` -> `Worker A`, `N_A` -> `Node A`, `G_1` -> `Global Owner 1`.**

#### Problem 2: Object Borrower Failure

In the chained object borrowing case, the owner of an object is not aware of the indirect borrowers. If all direct borrower fails, the owner will consider the object out-of-scope and GC the object. Accessing the object on indirect borrowers will fail with an `ObjectLostError`.

**TODO: Add issue link.**

For example, the owner of the object `O_A` borrowed `O_A` to the worker `W_A` which is on node `N_A`, and the worker `W_A` borrowed `O_A` to the worker `W_B`. If the worker `W_A` dies, the object `O_A` will be deleted. Calling `ray.get(O_A)` on the worker `W_B` will fail with an `ObjectLostError`.

![image_1_the_object_borrower_failure](https://user-images.githubusercontent.com/11995469/164454286-6d7658c7-952b-4351-bd06-7404f49db919.png)

#### Problem 3: Loss of All Copies

If all copies of an object are lost due to node failures, trying to access the object on any other node will fail with an `ObjectLostError` because there's no way to recover the data of the object. If there's only one copy of the object, the failure of the node on which the only copy is stored will result in inaccessibility of the object in the cluster.

For example, if the only copy of the object `O_A` is on node `N_A`, the object will be unavailable if the node `N_A` dies.

### Proposed Design

#### How to solve the object owner failure problem?

##### Option 1: syncing reference table of owners to a highly available external storage.

We keep the in-memory reference table of owners in sync with a high-availability external storage. When an owner restarts from failure, it can restore the reference table from the external storage and recreate the RPC connections with borrowers and Raylets.

**TODO: Mention that option 1 is still based on global owners, and reorganize if needed.**

**Pros**:

- Relatively simple.

**Cons**:

- Deployment is more complicated.
- The modification of the reference table is a high-frequency operation. Syncing with an external storage may hurt performance.
- Potentially some tricky consistency issues when an owner fails.

##### Option 2: dedicated global owners

A group of special Ray actors with `max_restarts=-1` will be created to own all objects of a job created by `ray.put`. When one of the special actors fails, the restarted instance will rebuild the reference table based on information provided by Raylets and other non-special workers.

**TODO: we should also support other types of objects, not just objects created by `ray.put`. e.g. task returns.**

These actors will be distributed onto different nodes with best effort.

A job-level configuration will be added (disabled by default) to enable this feature.

###### Detailed Design

There are two options, active way and passive way, to rebuild the reference table and the RPC connections when the global owner `G_1` restarts.

**The Active Way**:

The global owner actively collect information about the alive objects it owns to rebuild the reference table.

1. `G_1` begins to rebuild the reference table, and sets the rebuilding status to `REBUILDING`.
2. `G_1` sends RPCs to all Raylets and ask every Raylet to traverse all local workers and reply with the information of all objects owned by `G_1`. `G_1` then uses the information to rebuild the reference table and re-establish RPC connections.
3. `G_1` sets the rebuilding state to `READY`.
4. The reply callback of some RPCs (WaitForObjectFree) will not be called until the rebuilding state of `G_1` is set to `READY`.

**The Passive Way**:

As the following illustration shows, every Raylet maintains an RPC connection to every global owner to watch the health of the global owners. When the global owner `G_1` dies and the job is not finished yet,

1. Raylet will find out that `G_1` is dead through RPC disconnection.
2. When Raylet knows that `G_1` is restarted, Raylet sends the information below to `G_1`:
    - References of all objects which are owned by `G_1`. (Collected by traversing all local workers.)
    - Objects which are owned by `G_1` and are local (the data is in the local Object Store).
    - The spill URLs of locally spilled objects which are owned by `G_1`.
3. `G_1` will reconnect to the Raylet and the workers on it after receiving the above collected information from this Raylet.
4. `G_1` sets the state to `READY` after finished rebuilding reference table.
5. The reply callback of some RPCs (`WaitForObjectFree`) will not be called until the rebuilding state of `G_1` is set to `READY`.

**TODO: Which one is the accepted design? Active or passive?**

**API**:

```python
# Set the number of global owners, the default value is 0. The object store behaves the same as before when the number of global owners is 0.
ray.init(global_owner_number=16)
```

**TODO: The flag should be put into job config.**

![image_2_global_owner](https://user-images.githubusercontent.com/11995469/164455556-b2f4101b-23f4-46db-808a-4407d48526a6.png)

**Pros**:

- No need to rely on an external storage.
- Performance will not be affected.

**Cons**:

- **Active**: The rebuilding process may take a long time.
- **Passive**: When a global owner is waiting for collected information from Raylets, the global owner needs to handle timeouts, which can be difficult.
- Relatively complicated.

#### How to solve the object borrower failure problem?

##### Star Topology

We use star topology instead of tree topology when the number of global owners is greater than zero to make sure the owner directly lends an object to other workers.

As the following illustration shows, the worker `W_A` owns the object `O_A`, and the worker `W_B` have already borrowed `O_A` from `W_A`. Here is the process how `W_B` lends `O_A` to `W_C`

1. The worker `W_B` sends the object `O_A` to the worker `W_C`, and the object is stored in the reference table of `W_C`.
2. The worker `W_B` increments the reference count of object `O_A` by one to avoid the object being freed before finishing the lending process.
3. The worker `W_C` sends an async RPC to the worker `W_A`, so `W_A` adds `W_C` into `O_A`'s borrower list in `W_A`.
4. The worker `W_A` sends an async RPC (`WaitForRefRemoved`) to the worker `W_C`.
5. The worker `W_C` sends an async RPC to `W_A`.
6. The worker `W_B` decrements the reference count of object `O_A` by one on receiving the reply of the RPC in step 1.

![image_3_star_topology](https://user-images.githubusercontent.com/11995469/164456154-2163f505-d835-4d23-9901-fac6d867d368.png)

#### How to solve the loss of all copies problem?

##### Multiple Primary Copies

We make sure there are multiple (configurable) primary copies of the same object.

**TODO: Rewrite this section. Primary copy and backup copy -> multiple primary copies.**

**TODO: consider creating another copy once the object is sealed.**

**TODO: How to configure the count of primary copies? Maybe `ray.put(obj, _primary_copies=2)` and `ray.remote(_primary_copies=2)`?**

After the primary copy of an object is created, we create a backup of the object on another node. When the owner of object `O_A` finds out that the object's primary copy is unavailable, `G_1` will rebuild the primary copy with the following steps:

1. When the RPC connection disconnects, `G_1` will turn a backup copy into a primary copy and create a new backup copy.
2. `G_1` sends an RPC request to the Raylet which has a backup copy of `O_A` and turn it into a primary copy.
3. `G_1` sends an async RPC request to another Raylet to create a new backup.
4. The Raylet which has the new backup copy will keep an RPC connection with `G_1` to watch the health of `G_1`.

![image_4_rebuild_primary_copy](https://user-images.githubusercontent.com/11995469/164456515-0f9e7d15-51be-4bb4-8852-ca5017a0411e.png)

When `G_1` finds out that the backup copy of object `O_A` is unavailable, `G_1` will send an RPC request to the node `N_B` and ask it to create a new backup copy. As the illustration shows:

1. When the RPC connection disconnects, `G_1` finds out that the backup copy of `O_A` is unavailable.
2. `G_1` sends an RPC request to ask `N_B` to create a new backup copy.
3. `G_1` sends an RPC request to `N_C`.
4. `N_C` creates a backup copy of `O_A`.
5. `N_C` keeps an RPC connection with `G_1` to watch the health of `G_1`.

![image_4_rebuild_backup](https://user-images.githubusercontent.com/11995469/164456742-c585aea8-4df8-47a4-b632-e7b61c756536.png)

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

None.
