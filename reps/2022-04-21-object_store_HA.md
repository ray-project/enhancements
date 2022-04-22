## Summary
### General Motivation

Ray is a general-purpose and powerful computing framework, combined with the Object Store module, it can be easily extended into a data service to provide data for various machine learning tasks. Here are two more typical scenarios.

#### Scenarios One: Data-Provider for Deep Learning Offline Training

A data-provider can be implemented by Ray, which has the following functions:
1. Read data from various data sources and store them in the Object Store, which can be read by ray-client.
2. Has simple data processing capabilities, such as data augmentation, data distillation, etc.

#### Scenarios Two: Mars On Ray

[mars](https://github.com/mars-project/mars) is a tensor-based unified framework for large-scale data computation which scales numpy, pandas, scikit-learn and many other libraries.

Mars on Ray uses Ray actors as execution backend, Ray remote function-based execution backend is in progress. Mars will store data in the Ray Object Store.

#### Why does the Object Store need High availability?

In those scenario, there are two requirements for the service:
1. **Stability**: As a long-running service, it needs to be automatically restored after some process failure. Currently, any worker who dies can result in objects lost.
2. **Auto-Scaling**: Primarily reducing cluster size when data requirements are reduced. Drop any node can cause many objects to lost primary-copy, and those objects will become unavailable

### Goal

Any alive worker can read any object in scope via `ray.get`, when any node `N_A` in the ray cluster is dead.

### Should this change be within `ray` or outside?

main `ray` project. Changes are made to Ray Core level.

## Stewardship
### Required Reviewers

@stephanie-wang, @ericl, @scv119

### Shepherd of the Proposal (should be a senior committer)

@ericl

## Design and Architecture

### Problem statement

#### Problem One: The Object Owner Failure

The object owner stores the metadata of an object, it manages the reference and primary-copy of the object.

For example: If the owner of the object `O_A` in node `N_A`, the object `O_A` will be out of scope when node A dead. The primary copy will be deleted, and any worker who wants to get `O_A` will be told the object is lost.

#### Problem Two: The Object Borrower Failure

Borrower failure causes object reference loss,  and makes the owner release the object early.

For example: If the owner of the object `O_A` borrows `O_A` to the worker `W_A` which in node `N_A` and `W_A` borrow `O_A` to a new worker `W_B`, the object `O_A` will be released, when the worker `W_A` is dead. When calling `ray.get` on `W_B` will get an exception: RayObjectLostError.

![image_1_the_object_borrower_failure](https://user-images.githubusercontent.com/11995469/164454286-6d7658c7-952b-4351-bd06-7404f49db919.png)

#### Problem Three: The Primary Copy Lost

Primary copy loss due to node failure, will cause the object which does not have a copy on another node’s plasma store not to be available.

For example: If the primary copy of the object `O_A` in node `N_A` and `O_A` does not have any copy on another node’s plasma store, the object will be not available when the node `N_A` is dead.

### Proposed Design

#### How to solve owner failure?

##### Option one: Highly Available Storage to Store Object Metadata

Use high-availability storage to store original data to help owners restore.Store A in external storage to ensure data is not lost. When the owner restarts, the data in the `reference table` can be restored, and rebuild the RPC link.

**Pros**:

- Relatively simple.

**Cons**:

- Increased requirements for deployment environments.
- The modification of the `reference table` is a high frequency operation and may have performance issues.
- Potentially some trickier consistency issues when the owner fails.

##### Option two: Global Owner

Add a job level setting (default is disable) to enable this feature. Use ray-actor to be the global owner to own all objects created by interface `ray.put`. The global owner is a dummy actor with `max_restarts=-1`, and try to disperse on each node.

There were two options active or passive to rebuild the `reference table` and the RPC link  when `G_1` retry.

**Active**:

The global owner  will actively collect information about the alive objects it owns to rebuild the `reference table`.
1. `G_1` begins to rebuild the `reference table`, setting status to be `REBUILDING`.
2. Send RPC to all Raylets, ask every raylet to traverse all workers on it, and reply with the information of all objects owned by `G_1`. Use those information to rebuild the `reference table` and the RPC link.
3. `G_1` sets the state to `READY`.
4. Some RPC's (WaitForObjectFree) reply-callback will not be called before the state of `G_1` not `READY`.

**Passive**:

As the illustration shows, every raylet will maintain a RPC to every global owner, to watch those global owners. When the global owner `G_1` died and the job was not finished,

1. Raylet will find `G_1` dead through RPC disconnection, then traverse all the workers on current raylet, collect the information (reference, primary copy address) of all objects whose owner is `G_1`, and send it to `G_1` when rebuilding the RPC link.
2. `G_1` will reconnect to the raylet and the workers on it after receiving the RPC from this raylet.
3. `G_1` sets the state to `READY` after rebuilding with all raylets.
4. Some RPC's (WaitForObjectFree) reply-callback will not be called before the state of `G_1` not `READY`.

**API**:
```python
# Set the number of global owners, the default number is zero. The object store behaves the same as before when the number of global owners is zero.
ray.init(global_owner_number=16)
```
![image_2_global_owner](https://user-images.githubusercontent.com/11995469/164455556-b2f4101b-23f4-46db-808a-4407d48526a6.png)

**Pros**:

- No need to rely on external storage.
- Performance will not be affected.

**Cons**:

- Active: The rebuilding process may take a long time.
- **Passive**: When waiting for the RPC which is sent by raylets, the global owner needs to handle timeouts, which can be difficult.
- Relatively complex.

#### How to solve borrower failure?

##### Star Topology

Use Star Topology instead of Tree Topology, when the number of global owners is greater than zero, make the owner directly borrow the object to other workers.

As the illustration shows, the worker `W_A` owns the object `O_A`, and the worker `W_B` already borrows `O_A` from `W_A`. Here is the process by which `W_B` borrows `O_A` to `W_C`

1. The worker `W_B` sends the object `O_A` to the worker `W_C`, and it is stored in the `W_C` reference table.
2. The worker `W_B` increments the reference to object `O_A` by one. Avoid the object being freed before finish borrowed.
3. The worker `W_C` sends an async RPC to the worker `W_A`, making `W_A` add `W_C` in the borrowers list.
4. The worker `W_A` sends an async RPC(WaitForRefRemoved) to the worker `W_C`.
5. The worker `W_C` sends an async RPC to `W_A`.
6. The worker `W_B` reduces the reference to object `O_A` by one. When the RPC which sends in step 1 replies.

![image_3_star_topology](https://user-images.githubusercontent.com/11995469/164456154-2163f505-d835-4d23-9901-fac6d867d368.png)

#### How to solve primary copy loss?

##### The Backup of Primary Copy

After the primary copy is created, create a backup of it on another raylet. When the owner of object `O_A` finds its primary copy is unavailable, `G_1` will rebuild primary copy by fellow steps:
1. When RPC disconnects, it will turn backup to primary copy, and create a new backup.
2. Send a RPC request to the raylet which has `O_A` backup, turn it to primary copy.
3. `G_1` sends an async RPC request to another raylet to create a new backup.
4. The raylet which has the new backup will keep a RPC connected to watch `G_1`.

![image_4_rebuild_primary_copy](https://user-images.githubusercontent.com/11995469/164456515-0f9e7d15-51be-4bb4-8852-ca5017a0411e.png)

When `G_1` finds the backup of object `O_A` is unavailable, `G_1` will send a RPC request to the Node `N_B`, and make it create a new Backup. As the illustration shows:
1. When RPC disconnects, `G_1` finds the backup of `O_A` is unavailable.
2. `G_1` sends a RPC request to make `N_B` create a new backup.
3. `G_1` sends a RPC request to `N_C`.
4. `N_C` creates the backup of `O_A`.
5. `N_C` keeps a RPC connected to watch `G_1`.

![image_4_rebuild_backup](https://user-images.githubusercontent.com/11995469/164456742-c585aea8-4df8-47a4-b632-e7b61c756536.png)

## Compatibility, Deprecation, and Migration Plan

Fully forward compatible, the behavior of the object store will same as usual when the high-available mode is disabled.

## Test Plan and Acceptance Criteria

We plan to use a ray job to test the object store HA. About this job:

1. Multiple node, and each node has two type actors: producer and consumer.
    - **producer**: Produce data and cache `ObjectRef`. Add or delete objects according to a certain strategy, for testing object gc.
    - **consumer**: Get the `ActorHandle` for producer via actor name, and borrower object randomly through `ray.get`.
2. Adjust data scale according to parameters, include:
    - Data size, proportion of plasma store.
    - The number of objects.

Acceptance criteria:

1. Performance reduction is acceptable when no process or node failure.
2. Any one raylet process of any workers processes failure, and the job will finish in the end.

## (Optional) Follow-on Work

