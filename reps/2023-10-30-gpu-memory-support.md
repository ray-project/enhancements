# Ray GPU Memory Support

## Summary

Design docummentation for supporting gpu-memory on ray core resource scheduler.

### General Motivation

Currently, `ray` support `num_gpus` to scheduler resource to tasks/actors, which then assign either `num_gpus` number of gpus or a fraction of a single gpu if `num_gpus < 1`. However, there are often use cases where users want to specify gpu by amount of memory. The current workaround is ray users require to specify the gpu they want to use and convert the gpu memory into fractional value w.r.t the specified gpu. This new scheduler design will allows users to directly scheduler gpu resources by amount of memory.

**Ray community users’s demand**:
https://github.com/ray-project/ray/issues/37574

https://github.com/ray-project/ray/issues/26929

https://discuss.ray.io/t/how-to-specify-gpu-resources-in-terms-of-gpu-ram-and-not-fraction-of-gpu/4128

https://discuss.ray.io/t/gpu-memory-aware-scheduling/2922/5

### Should this change be within `ray` or outside?

Inside `ray` project since we want to add new parameter `_gpu_memory` to the `ray` remote function.

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
# Request a fractional GPU with specified gpu_memory in megabytes.
# Mutually exclusive with num_gpus.
@ray.remote(_gpu_memory=1024) 
def task:
  …
```

When a Ray node is started, Ray will auto detect the number of GPUs and GPU memory of each GPU and set the GPU and _gpu_memory resources accordingly. Users also have the option to manually specify `_gpu_memory` resources as the sum of total gpu memory across all gpus in the node. The default value is `num_gpus` * total memory of the gpu type in the node.

```bash
ray start # auto detection

ray start –num_gpus=3 –gpu_memory=3000 # manual override, each gpu has 1000mb total memory
```

Note that GPU memory and compute unit is 1-1 conversion, means 20GB of gpu memory is equivalent to 0.5 fractional value of an `A100_40GB` gpu. So, for simplicity and consistency, ray doesn't allow users to specify both `num_gpus` and `_gpu_memory` in a single ray task/actor.

```python
# Request a fractional GPU both num_gpus and _gpu_memory is not allowed
@ray.remote(_gpu_memory=1024, num_gpus=0.5) # raise ValueError exception
def not_allowed_task:
  …

# Request a fractional GPU with specified num_gpus.
@ray.remote(num_gpus=0.5)
def num_gpus_task:
  …

# Request a fractional GPU with specified _gpu_memory.
@ray.remote(_gpu_memory=1024) 
def gpu_memory_task:
  …
```

Additionally, users can still specify which gpu type they want to use by specifying `accelerator_type`.

```python
# Request a fractional of A100 GPU with specified _gpu_memory
@ray.remote(_gpu_memory=1024, accelerator_type="NVIDIA_A100")
def nvidia_a100_gpu_task:
  …

# Requesting 30GB of gpu memory from a A10 GPU with 24GB of memory.
# Task won't be able to be scheduled.
@ray.remote(_gpu_memory=30 * 1024, accelerator_type="NVIDIA_TESLA_A10G")
def nvidia_a10_gpu_task:
  …
```

### Implementation

In the implementation phase of the project, we introduce a new `ResourceID` named `GPU_Memory` to specify the amount of GPU memory resources. `GPU_Memory` is treated as a GPU resource with a distinct representation, where the relationship is defined as `GPU` equals to `GPU_Memory` divided by the total memory of a single GPU in the node as GPU resources in the node is homogeneous. Despite their distinct representations, both `GPU` and `GPU_Memory` signify the same underlying resource, with the caveat that Ray currently supports only a single GPU.

The primary implementation entails the automatic detection of GPU memory during the initialization of a Ray cluster. Subsequently, when users specify resources using `ray.remote`, particularly the `gpu_memory` resource, Ray will calculate the `num_gpus` from the specified `gpu_memory` in the `ResourceRequest`. This calculation is carried out by a function called `ConvertRelativeResource`, which updates `ResourceSet` from the `ResourceRequest` relative to the resources in the assigned node. Additionally, we incorporate a round-up mechanism during the conversion to `num_gpus`, ensuring precision up to $10^{-4}$.

There are two pivotal options in the major implementation of this project, each with distinct characteristics:

#### Option 1:
In Option 1, we choose to store `GPU_Memory` as part of the NodeResource. This implementation ensures synchronization between GPU and `GPU_Memory`. During node feasibility checks and resource allocation, the `ConvertRelativeResource` function performs two conversions: calculating `gpu_memory` if `num_gpus` is specified and vice versa. 

```c++
// Example: Given a Node with total_single_gpu_memory = 110000 (in mb) with ResourceRequest.ResourceSet
// {
//     "gpu_memory": 1010,
// }
// which is then converted to equivalent GPU rounded up and recompute gpu_memory after roundup
// {
//     "GPU": 0.0092, (1010/100000 = 0.00918181818 the rounded up)
//     "gpu_memory": 1012, (0.0092 x 110000)
// }

const ResourceSet NodeResources::ConvertRelativeResource(
    const ResourceSet &resource) const {
  ResourceSet adjusted_resource = resource;
  double total_gpu_memory = this->total.Get(ResourceID::GPU_Memory()).Double() /
    this->total.Get(ResourceID::GPU()).Double(); // get single GPU total memory
  // convert gpu_memory to GPU
  if (resource.Has(ResourceID::GPU_Memory())) {
    double num_gpus_request = 0;
    double gpu_memory_request = resource.Get(ResourceID::GPU()).Double();
    if (total_gpu_memory > 0) {
      // round up to closes kResourceUnitScaling
      num_gpus_request =
          (resource.Get(ResourceID::GPU_Memory()).Double() / total_gpu_memory) +
          1 / static_cast<double>(2 * kResourceUnitScaling);
      // update gpu_memory after roundup
      gpu_memory_request = num_gpus_request * total_gpu_memory;
    }
    adjusted_resource.Set(ResourceID::GPU(), num_gpus_request);
    adjusted_resource.Set(ResourceID::GPU_Memory(), gpu_memory_request);
  } else if (resource.Has(ResourceID::GPU())) {
    double gpu_memory_request = 0;
    if (total_gpu_memory > 0) {
      gpu_memory_request =
          (resource.Get(ResourceID::GPU()).Double() * total_gpu_memory)
          + 0.5;
    }
    adjusted_resource.Set(ResourceID::GPU_Memory(), gpu_memory_request);
  }
  return adjusted_resource;
}
```

Pros:
- Enhanced Observability: Users can see the remaining GPU memory resources after roundup allocation, providing detailed insights.

Cons:
- Synchronization Overhead: Requires synchronization for both `GPU` and `GPU_Memory`, introducing an additional layer of complexity by updating both `GPU` and `GPU_Memory` for rounding up.
- Resource Duality: Users need to grasp that both resources, `GPU` and `GPU_Memory`, essentially denote the same underlying resource.

#### Option 2:
In Option 2, we opt to store only GPU and convert `gpu_memory` to GPU. This implementation involves saving metadata of the single node's `total_gpu_memory`. `ConvertRelativeResource` converts `gpu_memory` in `ResourceRequest` to `num_gpus` based on the single node's `total_gpu_memory` during node feasibility checks and resource allocation. 

```c++
// Example: Given a Node with total_single_gpu_memory = 110000 (in mb) with ResourceRequest.ResourceSet
// {
//     "gpu_memory": 1010 
// }
// which is then converted to equivalent GPU resource
// {
//     "GPU": 0.0092 (1010/110000 = 0.00918181818 the rounded up)
// }

const ResourceSet NodeResources::ConvertRelativeResource(
    const ResourceSet &resource) const {
  ResourceSet adjusted_resource = resource;
  // convert gpu_memory to GPU
  if (resource.Has(ResourceID::GPU_Memory())) {
    double total_gpu_memory = 0;
    if (this->labels.find("gpu_memory") != this->labels.end()) {
      // TODO: raise exception if this is not true
      total_gpu_memory = std::stod(this->labels.at("gpu_memory"));
    }
    double num_gpus_request = 0;
    if (total_gpu_memory > 0) {
      // round up to closes kResourceUnitScaling
      num_gpus_request =
          (resource.Get(ResourceID::GPU_Memory()).Double() / total_gpu_memory) +
          1 / static_cast<double>(2 * kResourceUnitScaling);
    }
    adjusted_resource.Set(ResourceID::GPU(), num_gpus_request);
    adjusted_resource.Set(ResourceID::GPU_Memory(), 0);
  }
  return adjusted_resource;
}
```

Pros:
- Simplified Resource Model: Better emphasizes to new users that Ray has a single GPU resource, simplifying the resource model.
- Straightforward Conversion: `GPU_Memory` is converted to GPU based on the single node's total_gpu_memory during node feasibility checks and resource allocation with the roundup logic applied underhood.

Cons:
- Limited Observability: `ray.available_resources()` only displays remaining GPU resources in terms of percentage, without specific amounts for `GPU_Memory`.
- Incompatibility with Heterogeneous GPUs: Doesn't work for heterogeneous GPUs in a single node, a limitation existing in Ray's current support.
