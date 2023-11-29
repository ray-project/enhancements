# Ray GPU Memory Support

## Summary

Enhance Ray fractional gpu support with gpu memory based scheduling.

### General Motivation

Currently, `ray` supports `num_gpus` to scheduler resource to tasks/actors, which then assign either `num_gpus` number of gpu ids to be used by the tasks/actors. Additionally, `ray` provides support fractional gpus allocation by specifying `num_gpus < 1` so a single GPU can be used to run multiple tasks. This works well if the cluster only has a single type of GPUs. However, imagining a cluster has both A100 40GB and A100 80GB GPUs, setting num_gpus to a fixed number doesn’t work that well: if we set to 0.1 then we will get 4GB if the scheduler picks A100 40GB but 8GB if the scheduler picks A100 80GB which is a waste of resource if the task only needs 4GB. We can also set accelerator_type to A100_40GB and num_gpus to 0.1 to make sure we get the exact amount of GPU memory we need but then the task cannot run on A100 80GB even if it’s free. Many users also have encountered this issue:

- https://github.com/ray-project/ray/issues/37574

- https://github.com/ray-project/ray/issues/26929

- https://discuss.ray.io/t/how-to-specify-gpu-resources-in-terms-of-gpu-ram-and-not-fraction-of-gpu/4128

- https://discuss.ray.io/t/gpu-memory-aware-scheduling/2922/5

- https://discuss.ray.io/t/automatic-calculation-of-a-value-for-the-num-gpu-param/7844/4

- https://discuss.ray.io/t/ray-train-ray-tune-ray-clusters-handling-different-gpus-with-different-gpu-memory-sizes-in-a-ray-cluster/9220

This REP allows users to directly schedule fractional gpu resources by amount of memory. In our example, if user specify `gpu_memory = 20GB`, then `ray` automatically convert the value to `num_gpus` depending on which nodes the request is assigned too. As example, if it's scheduled on A100 40GB node, then `num_gpus = 0.5`, otherwise if it's scheduled on A100 80GB node, then `num_gpus = 0.25`. As a result, user can schedule a fixed amount of GPU resources without depending on which types of GPUs the tasks/actos are scheduled to.

... issue with num gpus

**Ray community users’s demand**:

https://github.com/ray-project/ray/issues/37574

https://github.com/ray-project/ray/issues/26929

https://discuss.ray.io/t/how-to-specify-gpu-resources-in-terms-of-gpu-ram-and-not-fraction-of-gpu/4128

https://discuss.ray.io/t/gpu-memory-aware-scheduling/2922/5

https://discuss.ray.io/t/automatic-calculation-of-a-value-for-the-num-gpu-param/7844/4

https://discuss.ray.io/t/ray-train-ray-tune-ray-clusters-handling-different-gpus-with-different-gpu-memory-sizes-in-a-ray-cluster/9220

### Should this change be within `ray` or outside?

Inside `ray` project since we want to add new parameter `gpu_memory` to the `ray` remote function.

## Stewardship

### Required Reviewers
The proposal will be open to the public, but please suggest a few experienced Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@pcmoritz, @jjyao, @scv119

### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

@jjyao

## Design

### API
Users will be able to specify amount of gpu memory to their ray tasks/actors using `gpu_memory` on `ray.remote`. The specified `gpu_memory` will be the amount of gpu resources from a single gpu that will be allocated to users ray tasks/actors.

```python
# Request a fractional GPU with specified gpu_memory in bytes.
# Mutually exclusive with num_gpus.
@ray.remote(gpu_memory=1024 * 1024 * 1024) # 1 mb request
def task:
  …
```

When a Ray node is started, Ray will auto detect the number of GPUs and GPU memory of each GPU and set the GPU and gpu_memory resources accordingly. Users also have the option to manually specify `gpu_memory` resources as the sum of total gpu memory across all gpus in the node. The default value is `num_gpus` * total memory of the gpu type in the node.

```bash
ray start # auto detection

ray start –num_gpus=3 –gpu_memory=3000 * 1024 * 1024 * 1024 # manual override, each gpu has 1000mb total memory
```

Note that GPU memory and compute unit is 1-1 conversion, means 20GB of gpu memory is equivalent to 0.5 fractional value of an `A100_40GB` gpu. So, for simplicity and consistency, ray doesn't allow users to specify both `num_gpus` and `gpu_memory` in a single ray task/actor.

```python
# Request a fractional GPU both num_gpus and gpu_memory is not allowed
@ray.remote(gpu_memory=1024 * 1024 * 1024, num_gpus=0.5) # raise ValueError exception
def not_allowed_task:
  …

# Request a fractional GPU with specified num_gpus.
@ray.remote(num_gpus=0.5)
def num_gpus_task:
  …

# Request a fractional GPU with specified gpu_memory.
@ray.remote(gpu_memory=1024 * 1024 * 1024) 
def gpu_memory_task:
  …
```

Additionally, users can still specify which gpu type they want to use by specifying `accelerator_type`.

```python
# Request a fractional of A100 GPU with specified gpu_memory
@ray.remote(gpu_memory=1024 * 1024 * 1024 * 1024, accelerator_type="NVIDIA_A100")
def nvidia_a100_gpu_task:
  …

# Requesting 30GB of gpu memory from a A10 GPU with 24GB of memory.
# Task won't be able to be scheduled.
@ray.remote(gpu_memory=30 * 1024 * 1024 * 1024 * 1024, accelerator_type="NVIDIA_TESLA_A10G")
def nvidia_a10_gpu_task:
  …
```

#### Placement Group
TBD

### Implementation
The primary implementation entails the automatic detection of GPU memory during the initialization of a Ray cluster.

```python
class AcceleratorManager:
    # return 0 if accelerator is not GPU, 
    # else return total GPU memory of a single GPU
    def get_current_node_gpu_memory(self):
      ...  
``` 

Subsequently, we added another change within the scheduler to convert `gpu_memory` to `num_gpus` depending on which node the request got assigned to check if resource request is feasible in the node and allocate the resource.

```pseudocode
for node in node_list: 

++  def convert_relative_resources(resource_request, node): # option 1 implementation
++      if gpu_memory in resource_request: 
++          resource_request.num_gpus = roundup(gpu_memory / node.label["gpu_memory_per_gpu"] , 0.0001)
++          resource_request.gpu_memory = 0
++      return resource_request
++  convert_relative_resources(resource_request, node)

    if check_is_feasible(resource_request):
        allocation = TryAllocate(resource_request)
```

#### Resources API
We introduce a new `ResourceID` named `GPU_Memory` (`gpu_memory` in string) to specify the amount of GPU memory resources. `GPU_Memory` is treated as a GPU resource with a distinct representation, where the relationship is defined as `GPU` equals to `GPU_Memory` divided by the total memory of a single GPU in the node as GPU resources in the node is homogeneous. Despite their distinct representations, both `GPU` and `GPU_Memory` signify the same underlying resource, with the caveat that Ray currently only supports homogeneous GPU type for each node.

We opt to store only `GPU` in `NodeResource` and convert `gpu_memory` to GPU means `gpu_memory` is `ResourceRequest` only resource. This implementation involves saving metadata of the single node's `total_single_gpu_memory`. `ConvertRelativeResource` converts `gpu_memory` in `ResourceRequest` to `num_gpus` based on the single node's `total_single_gpu_memory` during node feasibility checks and resource allocation.

```python
# Suppose we have two nodes with GPU type A100 40GB and A100 80gb respectively
NodeResource(available={"GPU": [1,1,1]}, label={"gpu_memory_per_gpu": "40GB"})
NodeResource(available={"GPU": [1,1,1]}, label={"gpu_memory_per_gpu": "80GB"})


# gpu_memory request
task.options(gpu_memory="10GB")

# equivalent resource request when scheduled in Node 1
ResourceRequest({"GPU": 0.25})
# remaining resources in Node 1, check using ray.available_resources()
NodeResource(available={"GPU": [0.75,1,1]})

# equivalent resource request when scheduled in Node 2
ResourceRequest({"GPU": 0.125})
# remaining resources in Node 2, check using ray.available_resources()
NodeResource(available={"GPU": [0.875,1,1]})


# Roundup gpu_memory request
task.options(gpu_memory="10MB")

# equivalent resource request when scheduled in Node 1
ResourceRequest({"GPU": 0.00025})
# round up to nearest 10^{-4}
ResourceRequest({"GPU": 0.0003})
# remaining resources in Node 1, check using ray.available_resources()
NodeResource(available={"GPU": [0.9997,1,1]})

# equivalent resource request when scheduled in Node 2
ResourceRequest({"GPU": 0.000125})
# round up to nearest 10^{-4}
ResourceRequest({"GPU": 0.0002})
# remaining resources in Node 2, check using ray.available_resources()
NodeResource(available={"GPU": [0.9998,1,1]})
```

Pros:
- Simplified Resource Model: Better emphasizes to new users that Ray only have `GPU` to represent GPU resource, simplifying the resource model.
- Straightforward Conversion: `GPU_Memory` is converted to GPU based on the single node's total_single_gpu_memory during node feasibility checks and resource allocation with the roundup logic applied underhood.

Cons:
- Limited Observability: `ray.available_resources()` only displays remaining GPU resources in terms of percentage, without specific amounts for `GPU_Memory`.
- Incompatibility with Heterogeneous GPUs: Doesn't work for heterogeneous GPUs in a single node, a limitation existing in Ray's current support.

##### Alternatives NodeResources API
For the alternative resource implementtion, we store `GPU_Memory` as part of the `NodeResource`. This implementation ensures synchronization between GPU and `GPU_Memory`. During node feasibility checks and resource allocation, the `ConvertRelativeResource` function performs two conversions: calculating `gpu_memory` if `num_gpus` is specified and vice versa. 

```python
# Suppose we have two nodes with GPU type A100 40GB and A100 80gb respectively
NodeResource(available={"GPU": [1,1,1], "gpu_memory": ["40GB", "40GB", "40GB"]})
NodeResource(available={"GPU": [1,1,1], "gpu_memory": ["80GB", "80GB", "80GB"]}) 


# gpu_memory request
task.options(gpu_memory="10GB")

# equivalent resource request when scheduled in Node 1
ResourceRequest({"GPU": 0.25, "gpu_memory": "10GB"})
# remaining resources in Node 1, check using ray.available_resources()
NodeResource(available={"GPU": [0.75,1,1], "gpu_memory": ["30GB", "80GB", "80GB"]}) 

# equivalent resource request when scheduled in Node 2
ResourceRequest({"GPU": 0.125, "gpu_memory": "10GB"})
# remaining resources in Node 2, check using ray.available_resources()
NodeResource(available={"GPU": [0.875,1,1], "gpu_memory": ["70GB", "80GB", "80GB"]}) 


# Roundup gpu_memory request
task.options(gpu_memory="10MB")

# equivalent resource request when scheduled in Node 1
ResourceRequest({"GPU": 0.00025, "gpu_memory": "10MB"})
# round up to nearest 10^{-4}
ResourceRequest({"GPU": 0.0003, "gpu_memory": "12MB"})
# remaining resources in Node 1, check using ray.available_resources()
NodeResource(available={"GPU": [0.9997,1,1], "gpu_memory": ["39.988GB", "40GB", "40GB"]})

# equivalent resource request when scheduled in Node 2
ResourceRequest({"GPU": 0.000125, "gpu_memory": "10MB"})
# round up to nearest 10^{-4}
ResourceRequest({"GPU": 0.0002, "gpu_memory": "16MB"})
# remaining resources in Node 2, check using ray.available_resources()
NodeResource(available={"GPU": [0.9998,1,1], "gpu_memory": ["79.984GB", "80GB", "80GB"]})
```

Pros:
- Enhanced Observability: Users can see the remaining GPU memory resources after roundup allocation, providing detailed insights.

Cons:
- Synchronization Overhead: Requires synchronization for both `GPU` and `GPU_Memory`, introducing an additional layer of complexity by updating both `GPU` and `GPU_Memory` for rounding up.
- Resource Duality: Users need to grasp that both resources, `GPU` and `GPU_Memory`, essentially denote the same underlying resource.

#### Autoscaler
For the first option, we require to node label to store `gpu_memory_per_gpu`. However, currently label is not included in autoscaler as label-based node scheduling is yet to be supported for autoscaler. Therefore, for current solution, we introduce a new private resources `node:gpu_memory_per_gpu__` as a constant label value representing `gpu_memory_per_gpu` node label. Next, we also add `convert_relative_resources` function before in `_fits` and `_inplace_subtract` in `resource_demand_scheduler.py`

```python
def _convert_relative_resources(node, resources):
        adjusted_resources = resources.copy()
    if "gpu_memory" in resources:
        if "node:gpu_memory_per_gpu" not in node or node["node:gpu_memory_per_gpu"] == 0:
            return None
        adjusted_resources["GPU"] = (
            resources["gpu_memory"] / node["node:gpu_memory_per_gpu"]
        )
        del adjusted_resources["gpu_memory"]
    return adjusted_resources

def _fits(node: ResourceDict, resources: ResourceDict) -> bool:
    adjusted_resources = _convert_relative_resources(node, resources)
    if adjusted_resources is None:
        return False
    for k, v in adjusted_resources.items():
        ...

def _inplace_subtract(node: ResourceDict, resources: ResourceDict) -> None:
    adjusted_resources = _convert_relative_resources(node, resources)
    if adjusted_resources is None:
        return
    for k, v in adjusted_resources.items():
        ...
```

For the second option, there's no required addition of `node:gpu_memory_per_gpu_` since `GPU_Memory` is part of resources, but the `_convert_relative_resources` still required.

#### Placement Group
TBD