
## Summary - Plasma Alternative
### General Motivation
Ray is a general-purpose and powerful computing framework and makes it simple to scale any compute-intensive Python workload. By storing the objects in Plasma, data can be efficiently (0-copy) shared between Ray tasks. We propose to provide a plasma-like object store with more functionalities targeted for different situations (e.g. to better support [Kubernetes](https://kubernetes.io/)). In other words, we can simultaneously have multiple plasma object-store variants with different functionalities. (e.g. users can choose a variant at `ray.init()`).

[**[Vineyard]**](https://github.com/v6d-io/v6d) is also an in-memory immutable data manager that provides out-of-the-box high-level abstraction and zero-copy in-memory sharing for distributed data in big data tasks. It aims to optimize the end-to-end performance of complex data analytics jobs on cloud platforms. It manages various data structures produced in the job, and cross-system data sharing can be conducted without serialization/deserialization overheads by decoupling metadata and data payloads.

**Opportunities**: Vineyard can enhance the ray object store with more functionalities for modern data-intensive applications. To be specific, the benefits are four-fold:

- O1: Sharing data with systems out of Ray. Vineyard is a daemon in cluster that any system can put/get objects into/from it.
- O2: Composability. Objects will not be sealed twice when we create a new object from a existing one(e.g. add a new column for dataframe).
- O3: Zero copy in put/get. Objects can be directly created in object store without copying or serialization.
- O4: Reusable Routines (IO/Chunking/Partitions).

**At the first step, we only focus on the O1**. Specifically speaking, Plasma store is currently an internal component of raylet, it shares the same fate with raylet. For systems that are not integrated with ray, they can not share objects via Plasma, thus have to share them by filesystem which will inevitably introduce heavy overheads if their data are complex (e.g. serializing a graph cost a long time.). While vineyard serves as a daemon in kubernete cluster.

#### Key requirements:
- Provide the options for users to select a third-party object store at `ray.init()`.
- [O1] Provide a  Plasma-like client which forward the request to vineyard client. (vineyard server provide very similar functionality as plasma).

### Should this change be within `ray` or outside?
main `ray` project. Changes are made to Ray Core level.

## Stewardship
### Required Reviewers
The proposal will be open to the public, but please suggest a few experience Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@ericl, @rkooo567, @scv119, @jjyao, @MissiontoMars

### Shepherd of the Proposal (should be a senior committer)
@scv119, @rkooo567

## Design and Architecture

The change will mostly happen in plasma client.  We propose to: (1) first expose the interface of plasma client, this will not change any logic of raylet (2) then add a new client implementation for that interface (e.g. vineyard client). Since the vineyard provides similar APIs as Plasma, the client implementation here may just wrap the request and forward to vineyard server instead of Plasma server. (3) handle the spilling and callback of vineyard. (4) handle the cpp/python/java wrapper.

We are going to split our changes into 4 PRs to make them friendly for review.

**Step 1: Expose the interface (for third-party object store client)**. Define mutually agreed interfaces for a third-party object store to inherit.  (The `PlasmaClientImpl` inherits the `ClientImplInterface`.) By defining the required ability of a store client, allowing other object stores (.e.g vineyard) to serve like a plasma mock. In this step, we will not add or delete anything into raylet.

```c++

class ClientImplInterface{
 public:
  // connect to the store server
  virtual Status Connect(const std::string &store_socket_name,
                 const std::string &manager_socket_name, int release_delay = 0,
                 int num_retries = -1) = 0;
  // create a blob with given size. add the reference 
  virtual Status CreateAndSpillIfNeeded(const ObjectID &object_id,
                                const ray::rpc::Address &owner_address, int64_t data_size,
                                const uint8_t *metadata, int64_t metadata_size,
                                std::shared_ptr<Buffer> *data, plasma::flatbuf::ObjectSource source, int device_num = 0) = 0;
  // create a blob immediately for some clients that can't wait. add the reference
  virtual Status TryCreateImmediately(const ObjectID &object_id, int64_t data_size,
                              const uint8_t *metadata, int64_t metadata_size,
                              std::shared_ptr<Buffer> *data, plasma::flatbuf::ObjectSource source, int device_num) = 0;

  // get objects by object_ids. add the reference 
  virtual Status Get(const std::vector<ObjectID> &object_ids, int64_t timeout_ms,
             std::vector<ObjectBuffer> *object_buffers, bool is_from_worker) = 0;

  // notify the server that this client does not need the object. remove the reference
  virtual Status Release(const ObjectID &object_id) = 0;

  // check the existence
  virtual Status Contains(const ObjectID &object_id, bool *has_object) = 0;

  // Abort an unsealed object as if it was never created at all.
  virtual Status Abort(const ObjectID &object_id) = 0;

  // Seal the object, make it immutable.
  virtual Status Seal(const ObjectID &object_id) = 0;

  // Delete the objects only when (1) they exist, (2) has been sealed, (3) not used by any clients. or it will do nothing.
  virtual Status Delete(const std::vector<ObjectID> &object_ids) = 0;

  // Delete objects until we have freed up num_bytes bytes.
  virtual Status Evict(int64_t num_bytes, int64_t &num_bytes_evicted) = 0;

  // disconnect from the store server.
  virtual Status Disconnect() = 0;

  // query the capacity of the object store.
  virtual int64_t store_capacity() = 0;
  
  // get the current debug string from the plasma store server.
  std::string DebugString() = 0;
};
```

**Step 2**: **Vineyard implement the interface to provide plamsa-like put/get/delete.** Like Plasma, Vineyard also has client-side and server-side. The client slide provides Plasma-compatible APIs which facilitates us to wrap Plasma requests into Vineyard requests.  We will implement `VineyardClientImpl` in a separate file. Both `PlasmaClientImpl` and `VineyardClientImpl` inherit from the `ClientImplIterface`, which allow users to select the desired client(Plasma or Vineyard) at runtime.

**Step 3:**  **Make the behavior of VineyardClientImpl consistent with PlasmaClientImpl.** Vineyard server also provide simlar function like plasma server, it also has reference count, eviction, and lifecycle management. We will try our best to align the behavior of plasma server and vineyard server. We will maintain a special code path in vineyard to provide the same functionality as Plasma. The elusive point maybe the spilling, since it is a callback which requires the raylet resources and cannot be executed in another process. An intuitively solution is to implement a proxy to replace the PlasmaStoreRunner in vineyard mode, which waits the messages from vineyard server to trigger these callback.

**Step 4:** **python/cpp/jave wrappers.** Handle the python/cpp/jave wrappers. provide demos, examples, and docs.

### Example - Code

To share the objects with out-of-Ray systems, vineyard provides API to persist a Ray object into an object that external systems can recognize(e.g. vineyard object in this proposal). The conversion should be zero-copy. The conversion APIs are provided by vineyard. We add a new option `plasma_store_impl` for ray to specify underlying object store, we reuse the option `plasma_store_socket_name` to specify the IPC socket path in this case.
```python
# users get objects from ray then put (zero-copy) them into vineyard or vice versa.
import ray
import vineyard as v6d

ray.init(plasma_store_impl="vineyard", plasma_store_socket_name="/tmp/vineyard.sock")

ray_data = [1,2,3,4]
ray_object_ref = ray.put(y)
ray_object = ray.get(object_ref)

v6d_object_id = v6d.from_ray(ray_object_ref) # 0-copy
# now the object is visable to systems out of ray.
print(v6d.get(v6d_object_id)) # [1,2,3,4]
```

### Hidden Assumption/Dependency

The goal of  lifecycle management in local plasma store is to drop some not-in-use objects (evict or spill) when we we ran out of memory. Both plasma and vineyard have thier own lifecycle management, and their behavior may be different. We think that raylet logic should not rely on the underlying lifecycle management strategies  (e.g. different eviction policy).  We propose to define a **Invariant** here that both vineyard and plasma should agree. For example, the invariant will define when a object adds/remove its reference, when object can be evicted/spilled. In other word The Invariant defines the impact on lifecycle management in server-side when calling the above  `ClientImplInterface` API in client-side. Since most of concerns about the integration may happens in this part, we may need further discussions about these invariants.

#### Object Ref_count

The Ray keeps two type of "reference count", one is `ref_count` in plasma to track the local plasma object, and the other is the `reference_count` in core_worker to track the primary copy. In this proposal, we will not change the logic of `reference_count`, and the vineyard server will have its own `ref_count`.

**Invariant #1**: A object will increase its `ref_count` iff the `Get(...)` , `TryCreateImmediately(...)`or `CreateAndSpillIfNeeded(...)`  is invoked. ([Create](https://github.com/ray-project/ray/blob/bb4ff42eeca50a5b91dd569caafdd71bf771b66e/src/ray/object_manager/plasma/store.cc#L198), [Get](https://github.com/ray-project/ray/blob/bb4ff42eeca50a5b91dd569caafdd71bf771b66e/src/ray/object_manager/plasma/store.cc#L110))

**Invariant #2:** A object will decrease its `ref_count` iff the `release(...)` or `Disconnect(...)`is invoked. ([Release](https://github.com/ray-project/ray/blob/bb4ff42eeca50a5b91dd569caafdd71bf771b66e/src/ray/object_manager/plasma/store.cc#L277), [Disconnect](https://github.com/ray-project/ray/blob/bb4ff42eeca50a5b91dd569caafdd71bf771b66e/src/ray/object_manager/plasma/store.cc#L346))

#### Object eviction

As mentioned above, the eviction policy may be different in different store variants. Currently, the policies of Plasma and Vineyard are both based on LRU.

**Invariant #3:** Eviction only happens in an eviction request and create request. ([evict]( https://github.com/ray-project/ray/blob/bb4ff42eeca50a5b91dd569caafdd71bf771b66e/src/ray/object_manager/plasma/store.cc#L451), [create](https://github.com/ray-project/ray/blob/bb4ff42eeca50a5b91dd569caafdd71bf771b66e/src/ray/object_manager/plasma/object_lifecycle_manager.cc#L191))

**Invariant #4:** Servers can only evict the objects that are not needed by clients, (a.k.a.reference count = 0). specially, pinned object should never be evcited. ([AddtoCache](https://github.com/ray-project/ray/blob/bb4ff42eeca50a5b91dd569caafdd71bf771b66e/src/ray/object_manager/plasma/object_lifecycle_manager.cc#L160))

#### Object spilling

When creating a new object with insufficient memory even with evicting possible objects, the local object manager will try to spill objects to external storage. The code path of spilling is out of Plasma (via callback) thus we can reuse it in third-party object-store.

**[callbacks]** Today the spilling of plasma relies on the callback. A `PlasmaStoreRunner` is running on a thread within raylet. Triggering the callback will directly call the methods in raylet. As a separate process, a third-party object store is unrealistic to execute these callbacks. The plasma store will also provide an [API](https://sourcegraph.com/github.com/ray-project/ray@149d06442bd0e53010ead72e4a7c0620ee1c0966/-/blob/src/ray/object_manager/object_manager.cc?L164) to query if an object is spillable.

**[solution]**  We can keep these callbacks.  The third-party object store is another process instead of a thread within Ray , this means the third-party object should not be launched by raylet, when raylet starts, the third-party object store is already there. Since we have to process these callbacks in the ray-side. Thus we can introduce a `PlasmaProxy`(which is a thread in raylet) here to process the callbacks. 

We put the `PlasmaProxy` on a thread in Raylet instead of the `PlasmaStoreRunner`. When triggering the callback in the third-party object store, it will send a request to the `PlasmaProxy` via IPC thus it can call the methods in raylet like plasma does so. Maybe we do not require API changes in the current client interface (a third-party object store will not do spilling itself, it only triggers spilling with a given threshold. In such a situation, a third-party object store will send a request to `PlasmaProxy` to trigger the spilling callback).

The `PlasmaProxy` will share fate with raylet, if vineyard is running into issues, `PlasmaProxy` will complain that (we can use heartbeat here). I think ray can treat the `PlasmaProxy` as the third-party object store instance, it will throw exceptions just like the original plasma runner.

**Invariant #5:** The spilling only happens in a object creation.(we trigger spilling when memory usage > threshold([triggle](https://github.com/ray-project/ray/blob/bb4ff42eeca50a5b91dd569caafdd71bf771b66e/src/ray/object_manager/plasma/store.cc#L174))) ([spill](https://github.com/ray-project/ray/blob/bb4ff42eeca50a5b91dd569caafdd71bf771b66e/src/ray/object_manager/plasma/create_request_queue.cc#L97))

**Invariant #6:** The spilling only happens when eviction can not meet the requirement. ([create object will first trigger eviction](https://github.com/ray-project/ray/blob/bb4ff42eeca50a5b91dd569caafdd71bf771b66e/src/ray/object_manager/plasma/object_lifecycle_manager.cc#L191))

**Invariant #7:** The spilling only happens in primary objects.(the spill_objects_callback will handle the spillng).

**Invariant #8:** Shouldn't unpin objects when spilling is initiated.

#### Object Delete

**Invariant #9:** `Delete()`a object only when (1) it exists, (2) has been sealed, (3) not used by any clients. or it will do nothing.

We believe that if a third-party object store client can follow the above invariants when implementation the `ClientImplInterface`. Then the ray logic can be seamlessly built on top of the third-party object store.

#### Dupilicate IDs in multiple ray cluster.

The third-party object store should not break the invariants within Ray. e.g when we have multiple ray clusters all connect to the third-party object store (there could be lots of duplicated object ids).

**[with visbility]** For vineyard-like third-party object store which has session/visibility/isolation mechanism, we can open a new session for each raylet (it is ok to have duplicated object ids which are seperated in different sessions.)

**[w/o visbility]** For other third-party object stores which do not provide such isolation control. we can add prefix to the duplicated object ids to distinguish them (this should be handled in `xxxClinetImpl` which inherits the `ClientImplInterface`).

**Invariant #10**: when the raylet is restarted, the storage should guarantee the object id from the previous raylet should be isolated to the new one.

#### Failure Mode

The third-party objects' failure mode should be consistent with plasma: which means a it should behave the same as plasma when object-store/ralet crashes. We introduce `XXXPlasmaProxy` to handle the failure. The Third-party should provide internal mechanism to block the connection when `XXXPlasmaProxy` runs into issues.

**[Situation 1]** When raylet crashes, workers can not get objects from third-party object store. Since the third-party object store will know it via the heartbeat of `XXXPlasmaProxy`, then it will close the connection of related client (the same session of `XXXPlasmaProxy` ). This is a recoverable failure for vineyard, when the raylet restart, it can connect to the previous session.

**[Situation 2]** When the third-party store crashes, the worker will not able to use the objects. The `XXXPlasmaProxy` will also know the fate of third-party object store just like Situation 1. Since the `XXXPlasmaProxy` is running on a thread of raylet, it can complain just like plasma runner crashes. This is an unrecoverable failure for vineyard. Since Raylet can not control the fate of vineyard.

**[Situation 3]** When the `XXXPlasmaProxy` crashes, the third-party store will behave like the raylet crashes, it will close the connection, forbid the clients in the same session to get objects. the raylet will behave like the object store crashes, it will throw exception just like original plasma runner. This is a recoverable failure for vineyard, raylet can restart PlasmaProxy, then connect to the prevoius session.

#### Callback 

**Invariant #11**:  When an object is added the `object_added callback` should be called, `object_deleted callback` should be called when an sealed object is deleted. `object_spill_callback` should be called when spilling is triggered.

**[Overhead]** The cost of a callback depends on the underlying store implementation. The abstraction here can be compatible with different object store. For plasma, the `PlasmaProxy` will execute the callback via in-process function call, for standalone store service like vineyard, the `VineyardPlasmaProy` will execute the callback via IPC. Supporting the third-party object store will not incurs a performance hit for current coude path (plasma-version).

If the IPC of third-party object store really obstacles the performance of ray scheduler (e.g. obtain the memory usage of the plasma store, and today ray design adopts the direct function call is to avoid to obtain the stale memory usage). We can share meta between ray and third-party object store via SHM. For example, the third-party object store writes the memory usage to SHM after each creation/deletion, and the ray read the memory usage from SHM when required.

## Compatibility, Deprecation, and Migration Plan
An important part of the proposal is to explicitly point out any compability implications of the proposed change. If there is any, we should thouroughly discuss a plan to deprecate existing APIs and migration to the new one(s).

- Ray Core
  - (**Step 1**) Add a new option named `plasma_store_impl` to choose the underlying local object store.  
  - (**Step 1**) Add a new option named `plasma_store_impl` to not launch a plasma runner in third-party mode.
  - (**Step 1**) Add a new virtual class named `ClientImplInterface` for third-party object store to implement.
  - (**Step 2**) Add a new `VineyardClientImp` to implement the `ClientImplInterface` and follw the above invariants.
  - (**Step 3**) Add a new `PlasmaRunnerProxy ` as a helper to execute the callback of raylet, raylet can decide whether to launch `ObjectStoreRunner` or `PlasmaRunnerProxy` depending on the configuration.
- Ray API
  - (**Step 4**) Handle the python/cpp/jave wrappers. provide demos, examples, and docs.
- Deprecation 
  - Deprecate the apis that directly call plasma store method in raylet such like `ObjectManager::IsPlasmaObjectSpillable`.


## Test Plan and Acceptance Criteria
The proposal should discuss how the change will be tested **before** it can be merged or enabled. It should also include other acceptance criteria including documentation and examples. 

- Unit and integration test for the above invariants
- Documentation with representative workload, covered by CI.

## (Optional) Follow-on Work
- To provide ray with more functionality as described in [**Opportunities**]
  - O2: Composability. Objects will not be sealed twice when we create a new object form a existing one(e.g. add a new column for dataframe).
  - O3: Zero copy in put/get. Objects can be directly created in object store without copying or serialization.
  - O4: Reusable Routines (IO/Chunking/Partitions).
