# Ray GPU Memory Support

## Summary

Enhance Ray fractional GPU support with GPU memory based scheduling.

### General Motivation

Currently, `ray` supports `num_gpus` for scheduling, which assigns `num_gpus` number of GPUs to tasks/actors. Additionally, `ray` provides fractional GPU allocation by specifying `num_gpus < 1` so a single GPU can be used to run multiple tasks. This works well if the cluster only has a single type of GPUs. However, imagining a cluster has both A100 40GB and A100 80GB GPUs, setting num_gpus to a fixed number doesn’t work that well: if we set to 0.1 then we will get 4GB if the scheduler picks A100 40GB but 8GB if the scheduler picks A100 80GB which is a waste of resource if the task only needs 4GB. We can also set accelerator_type to A100_40GB and num_gpus to 0.1 to make sure we get the exact amount of GPU memory we need but then the task cannot run on A100 80GB even if it’s free. Many users also have encountered this issue:

- https://github.com/ray-project/ray/issues/37574

- https://github.com/ray-project/ray/issues/26929

- https://discuss.ray.io/t/how-to-specify-gpu-resources-in-terms-of-gpu-ram-and-not-fraction-of-gpu/4128

- https://discuss.ray.io/t/gpu-memory-aware-scheduling/2922/5

- https://discuss.ray.io/t/automatic-calculation-of-a-value-for-the-num-gpu-param/7844/4

- https://discuss.ray.io/t/ray-train-ray-tune-ray-clusters-handling-different-gpus-with-different-gpu-memory-sizes-in-a-ray-cluster/9220

This REP allows users to directly schedule fractional GPU resources by amount of GPU memory. In our design, if a user specifies `gpu_memory = 20GB`, then `ray` automatically converts the value to `num_gpus` depending on which node the request is assigned to. As example, if it's scheduled on A100 40GB node, then `num_gpus = 0.5`, otherwise if it's scheduled on A100 80GB node, then `num_gpus = 0.25`. As a result, user can schedule a fixed amount of GPU resources without depending on which types of GPUs the tasks/actos are scheduled to.

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
Users will be able to specify amount of GPU memory to their ray tasks/actors using `gpu_memory` on `ray.remote`. The specified `gpu_memory` will be the amount of GPU resources from a single GPU that will be allocated to users ray tasks/actors.

```python
# Request a fractional GPU with specified gpu_memory in bytes.
# Mutually exclusive with num_gpus.
@ray.remote(gpu_memory=1024 * 1024 * 1024) # 1 mb request
def task:
  …
```

When a Ray node is started, Ray will auto detect the number of GPUs and GPU memory of each GPU. Users also have the option to manually specify them:

```bash
ray start # auto detection

ray start –num_gpus=3 –gpu_memory=1000 * 1024 * 1024 * 1024 # manual override, each GPU has 1000mb memory
```

Note that GPU memory and GPU is 1-1 conversion, means 20GB of GPU memory is equivalent to 0.5 fractional value of an `A100_40GB` GPU. So, for simplicity and consistency, ray doesn't allow users to specify both `num_gpus` and `gpu_memory` in a single ray task/actor.


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

Additionally, users can still specify which GPU type they want to use by specifying `accelerator_type`.

```python
# Request a fractional of A100 GPU with specified gpu_memory
@ray.remote(gpu_memory=1024 * 1024 * 1024 * 1024, accelerator_type="NVIDIA_A100")
def nvidia_a100_gpu_task:
  …

# Requesting 30GB of GPU memory from a A10 GPU with 24GB of memory.
# Task won't be able to be scheduled.
@ray.remote(gpu_memory=30 * 1024 * 1024 * 1024 * 1024, accelerator_type="NVIDIA_TESLA_A10G")
def nvidia_a10_gpu_task:
  …
```

#### Placement Group
User is also able to request `gpu_memory` in placement group bundles as follows:

```python
pg = placement_group([{"gpu_memory": 1024 * 1024, "CPU": 1}, {"GPU": 1}])
```

### Implementation
The primary implementation entails the automatic detection of GPU memory during the initialization of a Ray cluster.

```python
class AcceleratorManager:
    # return 0 if accelerator is not GPU, 
    # else return total GPU memory of a single GPU
    def get_current_node_gpu_memory(self):
      ...  
```
The detected GPU memory is added as a node label. 

During scheduling, the resource request that contains `gpu_memory` will be converted to the corresponding `num_gpus` resource request depending on which node the scheduler is considering.

```pseudocode
for node in nodes: 
  def convert_relative_resources(resource_request, node):
      if gpu_memory in resource_request: 
          resource_request.num_gpus = roundup(gpu_memory / node.label["gpu_memory_per_gpu"] , 0.0001)
          resource_request.gpu_memory = 0
      return resource_request
 
  resource_request = convert_relative_resources(resource_request, node)
 
  # After converting from gpu_memory to num_gpus, the remaining is the same  
  if check_is_available(resource_request, node):
      allocation = allocate(resource_request, node)
      break
```

```python
# Suppose we have two nodes with GPU type A100 40GB and A100 80gb respectively
NodeResource(available={"GPU": [1,1,1]}, label={"gpu_memory_per_gpu": 40GB})
NodeResource(available={"GPU": [1,1,1]}, label={"gpu_memory_per_gpu": 80GB})


# gpu_memory request
task.options(gpu_memory=10GB)

# equivalent resource request when scheduled in Node 1
ResourceRequest({"GPU": 0.25})
# remaining resources in Node 1
NodeResource(available={"GPU": [0.75,1,1]})

# equivalent resource request when scheduled in Node 2
ResourceRequest({"GPU": 0.125})
# remaining resources in Node 2
NodeResource(available={"GPU": [0.875,1,1]})


# Roundup gpu_memory request
task.options(gpu_memory="10MB")

# equivalent resource request when scheduled in Node 1
ResourceRequest({"GPU": 0.00025})
# round up to nearest 10^{-4} due to the precision limitation of FixedPoint
ResourceRequest({"GPU": 0.0003})
# remaining resources in Node 1
NodeResource(available={"GPU": [0.9997,1,1]})

# equivalent resource request when scheduled in Node 2
ResourceRequest({"GPU": 0.000125})
# round up to nearest 10^{-4}
ResourceRequest({"GPU": 0.0002})
# remaining resources in Node 2
NodeResource(available={"GPU": [0.9998,1,1]})
```

Pros:
- Simplified Resource Model: There is still only GPU node resource, gpu_memory is just another way to request the same GPU node resource.
- Straightforward Conversion: `gpu_memory` is converted to `num_gpus` based on the target node's GPU memory during scheduling.

Cons:
- Incompatibility with Heterogeneous GPUs: Doesn't work for heterogeneous GPUs in a single node, a limitation existing in Ray's current support.

#### Alternatives

For one alternative, we can store `GPU_Memory` as node resource and `num_gpus` resource request will be converted to `gpu_memory` resource request during scheduling but this is a bigger and breaking change compared to the proposed option.

Another alternative is having both `GPU_Memory` and `GPU` as node resources. But since they denote the same underlying resource, this modeling adds more confusion and the implementation needs to make sure these two node resources are synchronized (i.e. requesting one will also subtract the other).

#### Autoscaler
Autoscaler change is similar. We need to convert `gpu_memory` resource demand to `num_gpus` resource demand when considering each node type. Concretely, we also add `convert_relative_resources` function before in `_fits` and `_inplace_subtract` in `resource_demand_scheduler.py`:

```python
def _convert_relative_resources(node, resources):
    adjusted_resources = resources.copy()
    if "gpu_memory" in resources:
        adjusted_resources["GPU"] = (
            resources["gpu_memory"] / node.labels["gpu_memory_per_gpu"]
        )
        del adjusted_resources["gpu_memory"]
    return adjusted_resources

def _fits(node: ResourceDict, resources: ResourceDict) -> bool:
    adjusted_resources = _convert_relative_resources(node, resources)
    for k, v in adjusted_resources.items():
        ...

def _inplace_subtract(node: ResourceDict, resources: ResourceDict) -> None:
    adjusted_resources = _convert_relative_resources(node, resources)
    for k, v in adjusted_resources.items():
        ...
```

