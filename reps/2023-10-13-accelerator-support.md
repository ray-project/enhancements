# Ray Accelerator Support

## Summary

Holistic design of supporting accelerators in Ray.
This also serves as the reference doc for contributors
to add new accelerator support in the future.

### General Motivation

Nowadays, more and more specialized accelerators (e.g. TPU, HPU) are coming out to speed up AI workloads.
Supporting those accelerators natively in Ray will increase Ray's adoption as the computing framework for scaling AI applications.

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
Users will use accelerators through Ray resources. For GPU accelerators (e.g. Nvidia, AMD, Intel GPUs), they will use the `GPU` resource name. For other accelerators, they will each have their own resource name (e.g. TPU accelerator has resource name `TPU`). `accelerator_type` option can also be used to specify a particular type of accelerator for a task or actor.

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

Similar to `num_gpus`, having a top level parameter for each accelerator family:

```python
@ray.remote(num_tpus=1)
def task():
    ...

@ray.remote(num_tpus=1, accelerator_type="TPU-V4")
def task():
    ...
```

A single `ACCELERATOR` resource for the ability to schedule a task on any accelerator
and an `accelerator_family` is also introduced to allow specifying the family of the accelerators to use:

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

#### Implementation

Core defines an `Accelerator` base class and each accelerator needs to implement a subclass (e.g. `TPUAccelerator`):

```python
@DeveloperAPI
class Accelerator(ABC):
    @staticmethod
    @abstractmethod
    def is_available() -> bool:
        """Detect if the accelerator is available on this node."""

    @staticmethod
    @abstractmethod
    def get_resource_name() -> str:
        """Get the name of the resource representing this accelerator."""

    @staticmethod
    @abstractmethod
    def get_visible_accelerator_ids_env_var() -> str:
        """Get the env var that sets the ids of visible accelerators."""

    @staticmethod
    @abstractmethod
    def get_num_accelerators() -> int:
        """Get the total number of accelerators on this node."""

    @staticmethod
    @abstractmethod
    def get_visible_accelerator_ids() -> Optional[List[str]]:
        """Get the ids of accelerators visible to Ray."""

    @staticmethod
    @abstractmethod
    def get_accelerator_type() -> Optional[str]:
        """Get the type of the accelerator on this node.

        The result should only be used when get_num_accelerators() > 0.

        Return None if it's unknown.
        """

    @staticmethod
    @abstractmethod
    def validate_resource_request_quantity(quantity: float) -> Optional[str]:
        """Validate the resource request quantity of this accelerator resource.

        Return error message if the quantity is invalid.
        """

    @staticmethod
    @abstractmethod
    def set_visible_accelerator_ids(ids: List[str]) -> None:
        """Set the ids of accelerators visible to the Ray task or actor."""

    @staticmethod
    def get_ec2_num_accelerators(instance_type: str, instances: dict) -> Optional[int]:
        """Get the number of accelerators of this aws ec2 instance type.

        Return None if it's unknown.
        """
        return None

    @staticmethod
    def get_ec2_accelerator_type(instance_type: str, instances: dict) -> Optional[str]:
        """Get the accelerator type of this aws ec2 instance type.

        Return None if it's unknown.
        """
        return None
```

Besides defining the accelerator subclass, the accelerator resource needs to be marked as unit instance resource in config `RAY_custom_unit_instance_resources`.

### Ray Train

Ray Train will provide 2 new APIs to support different accelerator types. 


#### 1. Specify Accelerator Type in `ScalingConfig`

Ray Core automatically detects the accelerator type, and treats it as an additional resource tag. Ray Train will therefore be able to launch training jobs on specific accelerator types. We add a new initialization argument `accelerator_type` for `ray.train.ScalingConfig`.


```python
# CPU
ScalingConfig(num_workers=4)

# GPU
# If no accelerator is specified, schedule workers on any GPU node
ScalingConfig(num_workers=4, use_gpu=True)

# GPU (A100)
# If specified, schedule workers only on A100 nodes
ScalingConfig(num_workers=4, use_gpu=True, accelerator_type="A100")

# AWS Trainium
ScalingConfig(num_workers=4, resources_per_worker={"neuron_cores": 1})

# TPU (TPU-v3)
ScalingConfig(num_workers=4, accelerator_type="TPU-V3", resources_per_worker={"TPU": 1})

# HPU (Gaudi2)
ScalingConfig(num_workers=4, accelerator_type="Gaudi2", resources_per_worker={"HPU": 1})
```

Internally, Ray Train adds the `accelerate_type` parameter in `ray.remote()` to start the training workers.

#### 2. Specify PyTorch Backend for each Accelerator Family

Different accelerator family uses different backend to launch deep learning training. Ray Train will implement a different backend class for each accelerator family to launch distributed groups and set environment variables.

| Accelerator | Communication Backend | Developer APIs | User APIs | 
| - | - | - | - |
| GPU      | `nccl`, `gloo` | TorchBackend | `TorchConfig()` |
| TPU      | `xla[tpu]`      | TorchTPUXLABackend | `TorchConfig("xla[tpu]")` |
| Trainium | `xla[neuronx]`  | TorchAwsNeuronXLABackend | `TorchConfig("xla[neuronx]")`  |
| HPU      | `hccl`         | TorchHCCLBackend | `TorchConfig("hccl")` |


Putting things together, users should be able to set up the resources and backend with these two APIs in `TorchTrainer`.

```python
from ray.train import ScalingConfig
from ray.train.torch import TorchTrainer, TorchConfig

# TPU Training
trainer = TorchTrainer(
    train_func,
    scaling_config=ScalingConfig(
        num_workers=4, 
        accelerator_type="TPU-V4", 
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
TODO

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
ds.map_batches(Predicator, resources={"TPU": 1})
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
TODO

### Testing

For some of the CI tests, we can mock without actually running on the machine with those accelerators.
But for some other CI tests and release tests, we do need machines with those accelerators to make sure real workoads can run successfully using those accelerators.

