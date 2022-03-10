
## Summary - Plasma Alternative
### General Motivation
Ray is a general-purpose and powerful computing framework and makes it simple to scale any compute-intensive Python workload. By storing the objects in Plasma, data can be efficiently (0-copy) shared between Ray tasks. We propose to provide a plasma-like object store with more functionalities target for different situations (e.g. to better support [[Kubernetes](https://kubernetes.io/zh/)](https://kubernetes.io/zh/)). In other words, we can simultaneously have multiple plasma object-store variants with different functionalities. (e.g. users can choose a variant at `ray.init()`).

[**[Vineyard](https://github.com/v6d-io/v6d)**](https://github.com/v6d-io/v6d) is also an in-memory immutable data manager that provides out-of-the-box high-level abstraction and zero-copy in-memory sharing for distributed data in big data tasks. It aims to optimize the end-to-end performance of complex data analytics jobs on cloud platforms. It manages various data structures produced in the job, and cross-system data sharing can be conducted without serialization/deserialization overheads by decoupling metadata and data payloads.

**Opportunities**: Vineyard can enhance the ray object store with more functionalities for modern data-intensive applications. To be specific, the benefits are four-fold:

- O1: Sharing data with systems out of Ray. Vineyard is a daemon in cluster that any system can put/get objects into/from it.
- O2: Composability. Objects will not be sealed twice when we create a new object form a existing one(e.g. add a new column for dataframe).
- O3: Zero copy in put/get. Objects can be directly created in object store without copying or serialization.
- O4: Reusable Routines (IO/Chunking/Partitions).

**At the first step, we only focus on the O1**. Specifically speaking, Plasma store is currently an internal component of raylet, it shares the same fate with raylet. For systems that are not integrated with ray, they can not share objects via Plasma, thus have to share them by filesystem which will inevitably introduce heavy overheads if their data are complex (e.g. serializing a graph cost a long time.). While vineyard serves as a daemon in kubernete cluster.

#### Key requirements:
- Provide the options for users to select a third-party object store at `ray.init()`.
- [O1] Provide a  Plasma-like client which forward the request to vinyeard client. (vineyard server provide very similar functionality as plasma).

### Should this change be within `ray` or outside?
main `ray` project. Changes are made to Ray Serve level.

## Stewardship
### Required Reviewers
The proposal will be open to the public, but please suggest a few experience Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@ericl, @rkooo567, @scv119, @jjyao, @MissiontoMars

### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

TBF

## Design and Architecture

the change will mostly happen in plasma client. Here we may add a virtual class for specific store client to inherit. we propose to: (1) first expose the interface of plasma client, this will not change any logic of raylet (2) then add a new client implementation for that interface (e.g. vineyard client). Since the vineyard provides similar APIs as Plasma, the client implementation here may just wrap the request and forward to vineyard server instead of plasma server. (3) handle the spilling and callback of vineyard.

**Step 1: Expose the interface (for third-party object store client)**. Define a mutually agreed interface for a third-party object store to inherit. Make the `PlasmaClientImpl` to inherit the `ClientImplInterface`. By defining the required ability of a store client, allow other object stores (.e.g vineyard) to act like a plasma mock. In this step, we will not add or delete anything into raylet.

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
};
```

**Step 2**: **Vineyard implement the interface to provide plamsa-like put/get/delete.** Like Plasma, Vineyard also has client-side and server-side. The client slide provides Plasma-compatible APIs which facilitates us to wrap Plasma requests into Vineyard requests.  We will implement `VineyardClientImpl` in a separate file. Both `PlasmaClientImpl` and `VineyardClientImpl` inherit from the `ClientImplIterface`, which allow users to select the desired client(Plasma or Vineyard) at runtime.

**Step 3:**  **Make the behavior of VineyardClientImpl consistent with PlasmaClientImpl.** Vineyard server also provide simlar function like plasma server, it also has reference count, eviction, and lifecycle management. We will try our best to align the behavior of plasma server and vineyard server. We will maintain a special code path in vineyard to provide the same functionality as Plasma. The elusive feature maybe the spilling, since it is a callback which requires the raylet resources. An intuitively solution is to implement a proxy to replace plasmaRunner in vineyard mode, which waits the messages from vineyard server to trigger these callback.

**Step 4 :  python/cpp/jave wrappers. ** Handle the python/cpp/jave wrappers. provide demos, examples, and docs.

### Example - Code

To share the objects with out-of-Ray systems, vineyard provides API to persist a Ray object into an object that external systems can recognize(e.g. vineyard object in this proposal). The conversion should be zero-copy. The conversion APIs are provided by vineyard.
```python
# users get objects from ray then put (zero-copy) them into vineyard or vice versa.
import ray
import vineyard as v6d

ray.init(plasma_store_impl="vineyard")

ray_data = [1,2,3,4]
ray_object_ref = ray.put(y)
ray_object = ray.get(object_ref)

v6d_object_id = v6d.from_ray(ray_object_ref) # 0-copy
# now the object is visable to systems out of ray.
print(v6d.get(v6d_object_id)) # [1,2,3,4]
```

### Lifecycle Management

The goal of  lifecycle management in local plasma store is to drop some not-in-use objects (evict or spill) when we we ran out of memory. Both plasma and vineyard have thier own lifecycle management, and their behavior may be different. We think that raylet logic should not rely on the underlying lifecycle management strategies  (e.g. different eviction policy). Thus we should give a **protocol** here that both vineyard and plasma agree. For example, the protocol will define when a object add/remove its reference, when object can be evicted/spilled. In other word The protocol defines the impact on lifecycle management when calling the  `ClientImplInterface` API described above. Since most of concerns about the integration may happens in this part, we may need further discussions about these protocols.

#### Ref_count

The Ray keeps two type of "reference count", one is `ref_count` in plasma to track the local plasma object, and the other is the `reference_count` in core_worker to track the primary copy. In this proposal, we will not change the logic of `reference_count`, and the vineyard server will have its own `ref_count`.

**Protocol #1**: A object will increase its `ref_count` when the `Get(...)` or `CreateAndSpillIfNeeded(...)`  is invoked.

**Protocol #2:** A object will decrease its `refer_count` when the `release(...)` is invoked.

#### Object eviction

As mentioned above, the eviction policy may be different in different store variants. Currently, the policies of Plasma and Vineyard are both based on LRU.

**Protocol #3:** Servers can only evict the objects that are not needed by clients, (a.k.a.reference count = 0). specially, pinned object should never be evcited.

#### Object spilling

When creating a new object with insufficient memory even with evicting possible objects, the local object manager will try to spill objects to external storage. The code path of spilling is out of Plasma (via callback) thus we can reuse it in third-party object-store.

**Protocol #4:** The spilling only happens in a object creation. 

**Protocol #5:** The spilling only happens when eviction can not meet the requirement.

**Protocol #6:** The spilling only happens in pinned objects.

#### Object Delete

**Protocol #7:** `Delete()`a object only when (1) it exist, (2) has been sealed, (3) not used by any clients. or it will do nothing.

We believe that if a third-party object store client can follow the above protocol when implementation the `ClientImplInterface`. Then the ray logic can be seamlessly built on top of the third-party object store.

## Compatibility, Deprecation, and Migration Plan
An important part of the proposal is to explicitly point out any compability implications of the proposed change. If there is any, we should thouroughly discuss a plan to deprecate existing APIs and migration to the new one(s).

- Ray Core
  - Add a new option named `plasma_store_impl` to choose the underlying local object store.  
- Ray Serve 
  - Add a new option named `plasma_store_impl` to not launch a plasma runner in third-party mode.
  - Add a new virtual class named `ClientImplInterface` for third-party object store to implement.
  - Add a new `VineyardClientImp` to implement the `ClientImplInterface` and follw the above protocols.
  - Add a new `PlasmaRunnerProxy ` as a helper to execute the callback of raylet.
  
- Deprecation 
  - No API will be deprecated.


## Test Plan and Acceptance Criteria
The proposal should discuss how the change will be tested **before** it can be merged or enabled. It should also include other acceptance criteria including documentation and examples. 

- Unit and integration test for the above protocols
- Documentation with representative workload, covered by CI.

## (Optional) Follow-on Work
- To provide ray with more functionality as described in [**Opportunities**]
  - O2: Composability. Objects will not be sealed twice when we create a new object form a existing one(e.g. add a new column for dataframe).
  - O3: Zero copy in put/get. Objects can be directly created in object store without copying or serialization.
  - O4: Reusable Routines (IO/Chunking/Partitions).
