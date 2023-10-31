# Ray Accelerator Support

## Summary

Holistic design of supporting accelerators in Ray.
This also serves as the reference doc for contributors
to add new accelerator support in the future.

### General Motivation

Nowadays, more and more specialized accelerators (e.g. TPU, HPU) are coming out to speed up AI workloads.
Supporting those accelerators natively (similar to how Ray supports Nvidia GPUs) in Ray will increase Ray's adoption as the computing framework for scaling AI applications.

### Should this change be within `ray` or outside?

Inside `ray` project since we want to support those accelerators out of the box.

## Stewardship

### Required Reviewers
The proposal will be open to the public, but please suggest a few experienced Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@pcmoritz, @ericl, @scv119

### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

@scv119

## Design

### Ray Core

#### API
Users will use accelerators through Ray resources. For GPU accelerators (e.g. Nvidia, AMD, Intel GPUs), they will use the `GPU` resource name. For other accelerators, they will each have their own resource name (e.g. TPU accelerator has resource name `TPU`).

| Resource    | Accelerator               |
| ----------- | ------------------------- |
| GPU         | Nvidia, Intel, AMD, Apple |
| TPU         | TPU                       |
| HPU         | Habana Gaudi              |
| neuron_cores| AWS Trainium, Inferentia  |
| ...         | ...                       |

`accelerator_type` option can also be used to specify a particular type of accelerator for a task or actor.

```python
@ray.remote(num_gpus=1)
def task():
    ...

@ray.remote(num_gpus=1, accelerator_type="A100")
def task():
    ...

@ray.remote(num_gpus=1, accelerator_type="Intel-Max-1550")
def task():
    ...

@ray.remote(resources={"TPU": 1})
def task():
    ...

@ray.remote(resources={"TPU": 1}, accelerator_type="TPU-V4")
def task():
    ...
```

##### Alternatives
Other alternative APIs considered:

Similar to `num_gpus`, having a top level parameter for each accelerator family.
One issue is that this will add many top level parameters to `.remote()` that users
need to learn even though they don't use those accelerators.

```python
@ray.remote(num_tpus=1)
def task():
    ...

@ray.remote(num_tpus=1, accelerator_type="TPU-V4")
def task():
    ...
```

A single `ACCELERATOR` resource for the ability to schedule a task on any accelerator
and an `accelerator_family` is also introduced to allow specifying the family of the accelerators to use.
People feel this is less obvious or expclit than having a resource name per accelerator family
and the ability to schedule a task on any accelerator is not that needed.

```python
@ray.remote(resources={"ACCELERATOR": 1})
def task()
    ...

@ray.remote(resources={"ACCELERATOR": 1}, accelerator_family="TPU")
def task()
    ...

@ray.remote(resources={"ACCELERATOR": 1}, accelerator_type="TPU-V4")
def task()
    ...
```

Similar to the above but having a top level `num_accelerators` parameter:

```python
@ray.remote(num_accelerators=1)
def task()
    ...

@ray.remote(num_accelerators=1, accelerator_family="TPU")
def task()
    ...

@ray.remote(num_accelerators=1, accelerator_type="TPU-V4")
def task()
    ...
```

##### Runtime Context API

Currently we have `ray.get_gpu_ids()` and we need similar APIs for getting available ids of other accelerators.

The new API will be inside RuntimeContext since it returns accelerator ids that are available to a worker.

```python
class RuntimeContext:
    def get_accelerator_ids() -> Dict[str, List[str]]:
        """
        Get the current worker's available accelerator ids.

        Returns:
            A dictionary keyed by the resource name. The values are list
            of ids. E.g., `{'GPU': ['0', '1']}`, `{'neuron_cores': ['0', '3']}`.
        """
```

#### Implementation

Core defines an `AcceleratorManager` base class and each accelerator needs to implement a subclass (e.g. `TPUAcceleratorManager`):

```python
class AcceleratorManager(ABC):
  """This class contains all the functions needed for supporting
  an accelerator family in Ray."""

  @staticmethod
  @abstractmethod
  def get_resource_name() -> str:
      """Get the name of the resource representing this accelerator family.

      Returns:
          The resource name: e.g., the resource name for Nvidia GPUs is "GPU"
      """

  @staticmethod
  @abstractmethod
  def get_visible_accelerator_ids_env_var() -> str:
      """Get the env var that sets the ids of visible accelerators of this family.

      Returns:
          The env var for setting visible accelerator ids: e.g.,
              CUDA_VISIBLE_DEVICES for Nvidia GPUs.
      """

  @staticmethod
  @abstractmethod
  def get_current_node_num_accelerators() -> int:
      """Get the total number of accelerators of this family on the current node.

      Returns:
          The detected total number of accelerators of this family.
          Return 0 if the current node doesn't contain accelerators of this family.
      """

  @staticmethod
  @abstractmethod
  def get_current_node_accelerator_type() -> Optional[str]:
      """Get the type of the accelerator of this family on the current node.

      Currently Ray only supports single accelerator type of
      an accelerator family on each node.

      The result should only be used when get_current_node_num_accelerators() > 0.

      Returns:
          The detected accelerator type of this family: e.g., H100 for Nvidia GPU.
          Return None if it's unknown or the node doesn't have
          accelerators of this family.
      """

  @staticmethod
  @abstractmethod
  def validate_resource_request_quantity(
      quantity: float,
  ) -> Tuple[bool, Optional[str]]:
      """Validate the resource request quantity of this accelerator resource.

      Args:
          quantity: The resource request quantity to be validated.

      Returns:
          (valid, error_message) tuple: the first element of the tuple
              indicates whether the given quantity is valid or not,
              the second element is the error message
              if the given quantity is invalid.
      """

  @staticmethod
  @abstractmethod
  def get_current_process_visible_accelerator_ids() -> Optional[List[str]]:
      """Get the ids of accelerators of this family that are visible to the current process.

      Returns:
          The list of visiable accelerator ids.
          Return None if all accelerators are visible.
      """

  @staticmethod
  @abstractmethod
  def set_current_process_visible_accelerator_ids(ids: List[str]) -> None:
      """Set the ids of accelerators of this family that are visible to the current process.

      Args:
          ids: The ids of visible accelerators of this family.
      """

  @staticmethod
  def get_ec2_instance_num_accelerators(
      instance_type: str, instances: dict
  ) -> Optional[int]:
      """Get the number of accelerators of this family on ec2 instance with given type.

      Args:
          instance_type: The ec2 instance type.
          instances: Map from ec2 instance type to instance metadata returned by
              ec2 `describe-instance-types`.

      Returns:
          The number of accelerators of this family on the ec2 instance
          with given type.
          Return None if it's unknown.
      """
      return None

  @staticmethod
  def get_ec2_instance_accelerator_type(
      instance_type: str, instances: dict
  ) -> Optional[str]:
      """Get the accelerator type of this family on ec2 instance with given type.

      Args:
          instance_type: The ec2 instance type.
          instances: Map from ec2 instance type to instance metadata returned by
              ec2 `describe-instance-types`.

      Returns:
          The accelerator type of this family on the ec2 instance with given type.
          Return None if it's unknown.
      """
      return None
```

Besides defining the accelerator subclass, the accelerator resource needs to be marked as unit instance resource in config `RAY_custom_unit_instance_resources`.

### Ray Train

Ray Train will provide 2 new APIs to support different accelerator types. 


#### 1. Specify Resources in `ScalingConfig`

Ray Core automatically detects the accelerator type and the corresponding resources quantity. Ray Train users can directly specify the new accelerator resources in `ray.train.ScalingConfig(resources_per_worker=)`.


```python
# CPU
ScalingConfig(num_workers=4)

# GPU (Nvidia/Intel)
# If use_gpu=True, set resources_per_worker={"GPU": 1} by default
ScalingConfig(num_workers=4, use_gpu=True)

# AWS Trainium
ScalingConfig(num_workers=4, resources_per_worker={"neuron_cores": 1})

# TPU
ScalingConfig(num_workers=4, resources_per_worker={"TPU": 1})

# HPU
ScalingConfig(num_workers=4, resources_per_worker={"HPU": 1})
```

In the future, Ray Train will restructure the `ScalingConfig` class, and incorporate the `accelerate_type` parameter in a more structured way.

#### 2. Specify PyTorch Backend for each Accelerator Family

Different accelerator family uses different backend to launch deep learning training. Ray Train will implement a different backend class for each accelerator family to launch distributed groups and set environment variables.

| Accelerator | Communication Backend | Developer APIs | User APIs | 
| - | - | - | - |
| GPU      | `nccl`, `gloo` | TorchBackend | `TorchConfig()` |
| TPU      | `xla[tpu]`      | TorchTPUXLABackend | `TorchConfig("xla[tpu]")` |
| Trainium | `xla[neuronx]`  | TorchAwsNeuronXLABackend | `TorchConfig("xla[neuronx]")`  |
| HPU      | `hccl`         | TorchHCCLBackend | `TorchConfig("hccl")` |
| XPU      | `oneCCL`       | TorchOneCCLBackend | `TorchConfig("ccl")`|


Putting things together, users should be able to set up the resources and backend with these two APIs in `TorchTrainer`.

```python
from ray.train import ScalingConfig
from ray.train.torch import TorchTrainer, TorchConfig

# TPU Training
trainer = TorchTrainer(
    train_func,
    scaling_config=ScalingConfig(
        num_workers=4, 
        resources_per_worker={"TPU": 1}
    ),
    torch_config=TorchConfig(backend="xla[tpu]")
)

# AWS Trainium
trainer = TorchTrainer(
    train_func,
    scaling_config=ScalingConfig(
        num_workers=4, 
        resources_per_worker={"neuron_cores": 1}
    ),
    torch_config=TorchConfig(backend="xla[neuronx]")
)

# HPU
trainer = TorchTrainer(
    train_func,
    scaling_config=ScalingConfig(
        num_workers=4, 
        resources_per_worker={"HPU": 1}
    ),
    torch_config=TorchConfig(backend="hccl")
)
```


### Ray RLLib
Currently there is no plan to support other accelerators in RLLib.

### Ray Serve
Nothing needs to be changed on the serve side.

Users can define a deployment using accelerators via `ray_actor_options`:

```python
@serve.deployment(ray_actor_options={"num_gpus": 1})
class Deployment:
    ...

@serve.deployment(ray_actor_options={"resources": {"TPU": 1}})
class Deployment:
    ...
```

### Ray Data
Nothing needs to be changed on the data side.

Users can use the corresponding accelerator resources for data operations:

```python
ds.map_batches(..., resources={"TPU": 1})
```

### Kuberay

To auto populate `rayStartParams` resources using pod resources, some code is needed.
See https://github.com/ray-project/kuberay/blob/b7bc7ae8c983160bb700d6b8bb4014dda183ecea/ray-operator/controllers/ray/common/pod.go#L687 as an example of how GPU resource is populated.

### Observability

Currently we emit metrics about the physical usages of GPUs and also show them on Ray dashboard.
Similarly, for other accelerators, we need to emit those metrics as well and update the Ray dashboard
and Grafana dashboard to show them.

### Docker

Currently `rayproject/ray-ml` has `-gpu` versions that are based on `NVIDIA CUDA` image.
Similarly, for other accelerators, we need to publish docker images with their software installed.

### Documentation

All the supported accelerators need to be documented with code examples in both Ray core and Ray train.

### Testing

For some of the CI tests, we can mock without actually running on the machines with those accelerators.
But for some other CI tests and release tests, we do need machines with those accelerators to make sure real workoads can run successfully using those accelerators. Currently Ray CI only supports testing accelerators that are available on AWS.

