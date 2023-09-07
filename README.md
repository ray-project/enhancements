# Object Store Plugin Manager

## Summary
### General Motivation
Ray stores large objects in the Plasma distributed object store. The object store is implemented with shared memory, and multiple workers on the same node can reference the same copy of an object. On the other hand, if the worker’s local shared memory store does not yet contain a copy of the object, the object has to be replicated over with RPCs.  We propose object store plugin interfaces to allow ray to support other object stores that may provide additional features without disruption of the overall ray object management. For example, a CXL memory sharing based object store may avoid object replication entirely. 

### Should this change be within `ray` or outside?
Yes. The change should be within `ray`.

## Stewardship
### Required Reviewers
@ericl, @scv119

### Shepherd of the Proposal (should be a senior committer)

## Design and Architecture
The design consists of the object store plugin interface and the plugin manager.

### Plugin Interface

The plugin provides the functionalities of an object store responsible for storing and retrieving logic. It can be implemented as source code within the Ray repository or a shared library to be loaded dynamically. 

It shall implement classes inherited from `ObjectStoreClientInterface` and `ObjectStoreRunnerInterface`.   
`ObjectStoreClientInterface` replaces the current `PlasmaClientInterface`. It keeps all the functions from the current `PlasmaClientInterface` to retain full compatibility along with some extensions. Current `PlasmaClient` shall implement `ObjectStoreClientInterface`.

New virtual functions of `ObjectStoreClientInterface` :
```c++
/// Authentication to the object store.                                                     
virtual Status Authenticate(const std::string& user, const std::string& passwd) = 0;
virtual Status Authenticate(const std::string& secret) = 0;
/// The object storage optimized memory copy.
virtual void MemoryCopy(void* dest, const void* src, size_t len) = 0;
```

`ObjectStoreRunnerInterface` base class has all the functions from the current `PlasmaStoreRunner` to retain full compatibility along with some extensions. Current `PlasmaStoreRunner` shall implement `ObjectStoreRunnerInterface`.

Updated virtual function of `ObjectStoreRunnerInterface` with additional parameters:
```c++
// Pass additional startup parameters                                                       
virtual void Start(const std::map<std::string, std::string>& params, 
                   ray::SpillObjectsCallback spill_objects_callback,
                   std::function<void()> object_store_full_callback, 
                   ray::AddObjectCallback add_object_callback, 
                   ray::DeleteObjectCallback delete_object_callback) = 0;
```

New virtual functions of  `ObjectStoreRunnerInterface`:
```c++
/// The Current total allocated available memory size.
virtual int64_t GetTotalMemorySize() const = 0;

/// Maximal available memory size. It can be different from the Total memory size if the memory is dynamically expandable.
virtual int64_t GetMaxMemorySize() const = 0;
```

The diagram for current plasma object store classes:
![image](https://github.com/yiweizh-memverge/ray_plugin_manager_rep/blob/main/imgs/original_structure.png)

The diagram for proposed object store plugin classes:
![image](https://github.com/yiweizh-memverge/ray_plugin_manager_rep/blob/main/imgs/proposed_structure.png)

If the plugin is implemented as a shared library, the implementation should implement an API to create the object store client instance and the runner instance.
```c++
extern "C" { 
    ObjectStoreRunnerInterface CreateRunner(void); 
    ObjectStoreClientInterface CreateClient(void); 
}
```
### Plugin Manager

The plugin manager shall create the instances that implement `ObjectStoreClientInterface` and `ObjectStoreRunnerInterface` based on specified plugin name if the plugin is implemented within the Ray repository.  If a plugin object store is implemented as a POSIX dynamic linking library, the library shall be opened with its full path using `dlopen()`, `dlsym()` shall be used to look up symbols  for `CreateRunner()` and `CreateClient()` and to lazy create the instances when actually needed. 

The plugin manager shall provide the following APIs:
```c++
// Get an instance of the singleton PluginManager
Static PluginManager& GetInstance()

// Create the object store runner interface with the plugin object store name and configurations
std::unique_ptr<ObjectStoreRunnerInterface>
     CreateObjectorStoreRunnerInstance(const std::string& plugin_name,
   const std::string& plugin_path, 
   const std::string& plugin_config)

// Create the object store client interface with the plugin object store name and configurations
std::shared_ptr<ObjectStoreClientInterface> 
     CreateObjectorStoreClientInstance(const std::string& plugin_name,
                                       const std::string& plugin_path, 
                                       const std::string& plugin_config) 
```

The plugin object store can be specified during Ray startup. The following CLI parameters, or  `ray.init()` options are proposed for the plugin.
```c++
 plugin_name:       Optional[str] = ‘default’           
                    The name of the plugin object store. 
 plugin_path:       Optional[str] = ‘’                
                    The full path of the library if the plugin is a shared library.
 plugin_config:     Optional[Dict[str, any]] = None     
                    The objectstore startup configurations as a series of key-value pairs.  
```

The modified plasma plugin also would change the data path of how `CoreWorkerPlasmaStoreProvider` accesses the plasma storage. If global shared memory is used, it would retrieve from global shared memory, which appears as local memory.
```c++
CoreWorkerPlasmaStoreProvider::Get(&object_ids, …,results, ...) {
  /* check whether global shared Plasma object store is in use. */
  if (store_client_.IsGlobal()) {
    std::vector<ObjectID> obj_list;
    for (const auto& id : object_ids) {
      obj_list.emplace_back(id);
    }
    /*retrieve objects from global shared Plasma object store*/
    return GetIfLocal(obj_list, results); 
  }
  ...
}
```

## Compatibility, Deprecation, and Migration Plan
Currently this plugin manager doesn’t support native Windows platform, as Windows platform doesn’t have POSIX dlfcn support, which is used for loading shared libraries.

## Test Plan and Acceptance Criteria
The plugin manager will be fully unit tested. 

## (Optional) Follow-on Work
Optionally, the proposal should discuss necessary follow-on work after the change is accepted.
