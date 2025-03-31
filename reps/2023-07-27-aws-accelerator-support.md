# AWS accelerators support on Ray

## Background
*[Feel free to skip this section if you are already familiar with AI accelerator, AWS trn1 EC2 instance and NeuronCore]*

An AI Accelerator is a dedicated processor designed to accelerate machine-learning (ML) computations. These are
specialized hardware designed to improve performance, reduce latency and reduce cost of deploying ML based applications.

In late 2022, AWS announced general availability of Trn1 EC2 instances which are powered by AWS Trainium accelerators.
AWS Trainium accelerator is an AI accelerator, purpose built for high-performance deep learning (DL) training of 
generative AI models, including large language models (LLMs) and latent diffusion models[1].

Each Trainium accelerator (aka NeuronDevice) includes two second-generation NeuronCores(NeuronCore-v2). 
It is designed for speed model training. NeuronCore-v2 is a fully-independent heterogeneous compute-unit,
with multiple engines (Tensor/Vector/Scalar.. Engines), and on-chip memory, for maximizing data locality[2].
Also, Inferentia2 accelerator(inf2 which supports neuron-core) is designed to speed up inference.


## Summary

[Phase-1] Currently, Ray supports limited accelerators (only NVIDIA hardware) for GPUs which does not include
AWS Trainium/Inferentia2 accelerators.

[Phase-2] Also, Ray Train only supports Pytorch but not Pytorch-XLA (Accelerated Linear Algebra) which is a connector
between Pytorch deep learning framework and TPUs/NeuronCores.
Without these, customers can neither use AWS Trainium/Inferentia2 accelerators on Ray cluster by default nor use it for
distributed training on Ray Train.

## Stewardship

### Required Reviewers
@scv119 @matthewdeng

### Shepherd of the Proposal (should be a senior committer)
@scv119 @matthewdeng


## Design and Architecture
### Phase1

On Ray node initialization, each Raylet represents resource configuration with pre-defined resources
(CPU, GPU, object_store_memory...) and custom resources which resolves to resource specifications.
These node specifications are advertised to RayScheduler which will be used for work assignment.

Unlike distributed tasks. GPUs do not have Python interpreters. Instead of sending python lambdas, high-level
tools like Torch, Tensor will generate or call native GPU/accelerator code. CUDA and Neuron SDK are some low-level
libraries for interacting with GPUs/accelerators.

Currently, Ray supports/detects only NVIDIA accelerators. We make appropriate changes to make AWS accelerators visible
using Neuron-Runtime/Neuron SDK

```text
On node initialization:
if assigned_gpus:
    check NEURON_RT_VISIBLE_CORES <= assigned_gpus
else:
    auto_detect_number_of_neuron_cores and claim as GPU
Gather GPU type information if possible

On Raylet:
Reserve the neuron_cores to Raylet/WorkerNode by assigning the number
of neuron-cores based on assigned GPU
// For example let say, for 32 neuron-core machine (i-1) if we initialize
// the cluster with num_gpu=4, we would reserve [0, 1, 2, 3] neuron-cores
// to Raylet/WorkerNode

Lastly, add support for these accelerator_type on resources
and auto-scaling NodeProvisioner

```

### Phase2
Ray Train automatically sets up distributed process group and provides utility methods to prepare your model and data
for distributed training. Ray Train supports TorchTrainer for data parallel PyTorch training which supports
SPMD (single program, multiple data) paradigm. Each trainer/deep-learning framework is backed by a Backend which
is used for distributed communication between workers/actors.

TorchBackend is the communication for TorchTrainer and it supports limited backends (nccl, gloo) today.
In order to support NeuronCore we would use PythonXLA framework and configure the backend to XLA.
Also, requires additional configuration of torch-elastic (now called tourchrun) environment variables
for the XLA devices to detect.

```text
class _TorchBackend(Backend):
    def on_start():
        # support xla backend
        # Configure master env of xla device related to torchrun/torch-elastic
    def on_shutdown():
        # cleanup NeuronCore cache if needed
    def on_training_start():
        # configure rank/world_size/node_rank based on xla device



# Usage
trainer = TorchTrainer(
    train_func,
    train_loop_config=config,
    scaling_config=ScalingConfig(num_workers=2, use_gpu=True, resources_per_worker={"num_gpu": 1})
    ...
 )
```

### Should this change be within `ray` or outside?
1. For auto-detection, the changes are within RayCore
2. For XLA backend support, the changes are with RayTrain

### Compatibility
1. Able to auto-detect neuron cores as well as any existing accelerators
```text
2023-07-27 22:48:08,742 INFO worker.py:1621 -- Started a local Ray instance.
{'node:__internal_head__': 1.0, 'CPU': 8.0, 'memory': 18270223566.0, 'object_store_memory': 9135111782.0, 'node:172.31.55.43': 1.0, 'GPU': 2.0}
(GPUActor pid=345602) ray.get_gpu_ids(): [0]
(GPUActor pid=345602) rt_visible_cores: 0
(GPUActor pid=345602) {'logits': tensor([[-1.4126, -1.9890, -1.3332, -0.2176,  3.9735, -0.6969,  1.8381]])}
(use_gpu pid=345710) ray.get_gpu_ids(): [1]
```

### Deprecation, Migration Plan
Not required

### Test Plan and Acceptance Criteria
1. Add unit-test coverage for [Phase-1](#Phase1) auto-detection
2. Manual testing using real EC2 trn1 instance to validate the behavior

### Future implementation
Add support for other deep-learning trainers (Tensorflow...) on RayTrain as [NeuronSDK](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/index.html) support follows.

### Related Issues
* [33504](https://github.com/ray-project/ray/issues/33504)
* [33586](https://github.com/ray-project/ray/issues/33586)
