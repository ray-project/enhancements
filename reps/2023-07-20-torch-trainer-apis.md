# [Train] Unify Torch based Trainers on the `TorchTrainer` API

## Summary

Remove the `LightningTrainer`, `TransformersTrainer`, and `AccelerateTrainer` APIs and unify them on the `TorchTrainer` API.

### General Motivation

Ray Train currently exposes a different `Trainer` API for each individual training framework. The PyTorch based Trainers are as follows:

| Framework | Ray Train Trainer |
| ----------- | ----------- |
| PyTorch | `TorchTrainer` |
| Lightning | `LightningTrainer` |
| HuggingFace Transformers | `TransformersTrainer` |
| HuggingFace Accelerate | `AccelerateTrainer` |

While this pattern makes it very obvious to the user that Ray Train provides an integration with a given framework, we have observed a few drawbacks:
1. There is overhead for the user to learn and use these APIs.
    1. They are all different from each other.
    2. They are different from their corresponding framework.
    3. The user cannot see the internal training loop, which makes it hard to implement and debug their code.
2. The `LightningTrainer` and `TransformersTrainer` APIs are opionated and may not allow the user to fully express their desired training logic (e.g. validation and testing).

This proposal explores the idea of centralizing on a single `TorchTrainer` as the single way running training code for PyTorch-based frameworks in a distributed fashion.

This change is motivated by the following goals:
1. **Transparency:** Enables users to convert existing code to use Ray Train with minimal changes.
2. **Flexibility:** Allows users to easily leverage the full suite of functionality of their framework (e.g. `lightning.Trainer.test()`, `transformers.Trainer.predict()`) or even convert their code between frameworks (e.g. Transformers → Accelerate).
3. **Simplicity:** Reduces the surface area of the Ray Train interface, lowering overhead for both users and developers.

### Should this change be within `ray` or outside?

This change is part of the main `ray` project, which is where these Trainer APIs live as part of the Ray Train library.

## Stewardship
### Required Reviewers

@krfricke, @justinvyu, @woshiyyya

### Shepherd of the Proposal (should be a senior committer)

@ericl

## Design and Architecture

In today's architecture, `LightningTrainer`, `TransformersTrainer`, and `AccelerateTrainer` are all subclasses of `TorchTrainer`.

In the proposed design, we will remove this extra layer of abstraction, and users will be able to run code for their framework of choice directly through the `TorchTrainer`:

```python
from ray.train.torch import TorchTrainer

def train_func(config):
    # user Torch/Lightning/Transformers/Accelerate code goes here

trainer = TorchTrainer(train_func, ...)
trainer.fit()
```

**Reporting & Checkpointing:**

A critical part of this API change is ensuring that the user can report metrics as well as save and load checkpoints with their desired framework. In [[REP] Consolidated persistence API for Ray Train/Tune #35](https://github.com/ray-project/enhancements/pull/35), we introduce a simpler API for Checkpoints in which Ray Train will treat the contents of the Checkpoint as an opaque directory. As part of this current REP, we will show how we can utilize this new API along with the frameworks' native checkpointing APIs to fully support checkpointing, and remove the need for the existing `<Framework>Checkpoint` APIs:

- [TorchCheckpoint](https://docs.ray.io/en/releases-2.6.0/train/api/doc/ray.train.torch.TorchCheckpoint.html)
- [LightningCheckpoint](https://docs.ray.io/en/releases-2.6.0/train/api/doc/ray.train.lightning.LightningCheckpoint.html)
- [TransformersCheckpoint](https://docs.ray.io/en/releases-2.6.0/train/api/doc/ray.train.huggingface.transformers.TransformersCheckpoint.html)

To do this we rely on a few key utilty APIs.
```python
def train_func():
    # Restore checkpoint.
    checkpoint = ray.train.get_checkpoint()
    ...
    # Create checkpoint.
    checkpoint = ray.train.Checkpoint.from_directory(...)
    ray.train.report(metrics, checkpoint)
```

As an example, we can modify a Torch training script that is run with the `TorchTrainer` as follows:
```python
CHECKPOINT_FILE_NAME = "model.pt"

def train_func():
    # Restore checkpoint.
    checkpoint = ray.train.get_checkpoint()
    if checkpoint:
        checkpoint_dir = checkpoint.to_directory()
        checkpoint_path = Path(checkpoint_dir) / CHECKPOINT_FILE_NAME
        checkpoint_data = torch.load(checkpoint_path)
    ...
    # Create checkpoint.
    temp_dir = tempfile.TemporaryDirectory()
    checkpoint_path = Path(temp_dir) / CHECKPOINT_FILE_NAME
    torch.save(checkpoint_data, checkpoint_path)
    checkpoint = ray.train.Checkpoint.from_directory(temp_dir)
    ray.train.report(metrics, checkpoint)
```

In the following sections, we show how this change will be reflected each of the individual frameworks by comparing:
1. A (minimal) training script for the framework.
2. The training script rewritten using the current Ray Train APIs.
3. The training script rewritten using the proposed Ray Train APIs.
   1. Additionally, we show how to report metrics and handle checkpoints with Ray Train.

### Lightning

#### Lightning
```python 
import lightning

class MyLightningModule(lightning.LightningModule):
    ...

trainer = lightning.Trainer(**trainer_kwargs)
trainer.fit(MyLightningModule(**module_kwargs), **fit_kwargs)
```

#### `LightningTrainer` (Current)

In the existing `LightningTrainer`, we expose a `LightningConfigBuilder` that allows the user to configure the `lightning.Trainer` in a way that is compatible with Ray Train. While similar to the native Lightning interface, this requires a non-trivial amount of rewriting of the user's training code and does not provide a strict 1-1 mapping.

When `LightningTrainer.fit` is called, it will run an opinionated training loop that constructs the `lighting.Trainer` and calls `lightning.Trainer.fit()`.

```python
import lightning
from ray.train.lightning import (
    LightningTrainer, 
    LightningConfigBuilder
)

config_builder = LightningConfigBuilder()
config_builder.module(MyLightningModule, **module_kwargs)
config_builder.checkpointing(**checkpoint_kwargs)
config_builder.trainer(**trainer_kwargs)
config_builder.fit_params(**fit_kwargs)
lightning_config = config_builder.build()

trainer = LightningTrainer(
    lightning_config=lightning_config,
    ...
)
trainer.fit()
```

#### `TorchTrainer` (Proposed):

In this proposal, we provide a few common utilties that the user can use directly in their training code to configure the `Trainer` object.

- `get_devices` - Returns a list of devices to use for training.
- `prepare_trainer` - Validates and/or makes modifications so that the `Trainer` object is configured correctly to be compatible with Ray Train.
- `RayDDPStrategy`, `RayFSDPStrategy`, `RayDeepSpeedStrategy` - `LightningStrategy`s for different training strategies that are compatible with Ray Train.
- `RayLightningEnvironment` - A `LightningEnvironment` that is compatible with Ray Train.

With this, the user can directly interact with the Lightning interface, and is able define their training logic to use additional functionality such as `lightning.Trainer.test()`.

```python
import lightning
import ray.train.lightning
from ray.train.torch import TorchTrainer

def train_func(config):

    ...

    lightning_trainer = lightning.Trainer(
        devices=ray.train.lightning.get_devices(),
        strategy=ray.train.lightning.RayDDPStrategy(),
        plugins=[ray.train.lightning.RayLightningEnvironment()],
        **trainer_kwargs
    )
    lightning_trainer = ray.train.lightning.prepare_trainer(lightning_trainer)
    lightning_trainer.fit(MyLightningModule(**module_kwargs), **fit_kwargs)


trainer = TorchTrainer(
    train_loop_per_worker=train_func,
    ...
)
trainer.fit()
```

**Reporting & Checkpointing:**

For Lightning, we introduce a `RayTrainReportCallback`(`Callback`) that will report metrics and checkpoints to Ray Train.

```python
from lightning.callbacks.pytorch import Checkpoint

CHECKPOINT_FILE_NAME = "checkpoint.ckpt"

class RayTrainReportCallback(Checkpoint):
    # Define logic for calling `ray.train.report` as a Callback.
    ...

def train_func():
    # Create checkpoint.
    report_checkpoint_callback = RayTrainReportCallback()
    trainer = Trainer(..., callbacks=[report_checkpoint_callback])
    # Restore checkpoint.
    checkpoint_path = None
    checkpoint = ray.train.get_checkpoint()
    if checkpoint:
        checkpoint_dir = checkpoint.to_directory()
        checkpoint_path = Path(checkpoint_dir) / CHECKPOINT_FILE_NAME
    trainer.fit(model, ckpt_path=checkpoint_path, ...)
```

### HuggingFace Transformers

#### Transformers
```python 
import transformers

trainer = transformers.Trainer(**trainer_kwargs)
trainer.train()
```

**`TransformersTrainer` (Current):**

In the existing `TransformersTrainer`, we expose a thin `trainer_init_per_worker` function that allows the user to configure the `transformers.Trainer`.

When `TransformersTrainer.fit` is called, it will run an opinionated training loop that constructs the `transformers.Trainer` and calls `transformers.Trainer.train()`.

```python
import transformers
from ray.train.huggingface import TransformersTrainer

def trainer_init_per_worker(train_dataset, eval_dataset, **config):
    ...
    return transformers.Trainer(**trainer_kwargs)

trainer = TransformersTrainer(
    trainer_init_per_worker=trainer_init_per_worker,
    trainer_init_config=trainer_init_config,
    ...
)
trainer.fit()
```

#### `TorchTrainer` (Proposed)

In this proposal, we provide a `prepare_trainer` utility that the user can use directly in their training loop code to validate and/or make modifications so that the `Trainer` object is configured correctly to be compatible with Ray Train.

With this, the user is able define their training logic to use additional functionality such as `transformers.Trainer.evaluate()`.

```python
import transformers
import ray.train.huggingface.transformers
from ray.train.torch import TorchTrainer

def train_func(config):
    ...
    transformers_trainer = transformers.Trainer(**trainer_kwargs)
    transformers_trainer = ray.train.huggingface.transformers.prepare_trainer(transformers_trainer)
    transformers_trainer.train()

trainer = TorchTrainer(
    train_loop_per_worker=train_func,
    ...
)
trainer.fit()
```

**Reporting & Checkpointing:**

For Transformers, we introduce a `RayTrainReportCallback`(`TrainerCallback`) that will report metrics and checkpoints to Ray Train.

```python
from transformers.trainer_callback import TrainerCallback

class RayTrainReportCallback(TrainerCallback):
    # Define logic for calling `ray.train.report` as a Callback.
    ...

def train_func():
    ...
    trainer = Trainer(...)
    # Create checkpoint.
    report_checkpoint_callback = RayTrainReportCallback()
    trainer.add_callback(report_checkpoint_callback)
    # Restore checkpoint.
    checkpoint_path = None
    checkpoint = ray.train.get_checkpoint()
    if checkpoint:
        checkpoint_dir = checkpoint.to_directory()
        checkpoint_path = Path(checkpoint_dir) / CHECKPOINT_FILE_NAME
    trainer.train(resume_from_checkpoint=checkpoint_path)
```

### HuggingFace Accelerate

At its core, HuggingFace Accelerate can be used by configuring and instantiating an `Accelerator` object in your training code.

HuggingFace Accelerate also provides two **optional** CLI commands:
1. `accelerate config` - Generates a configuration file that is used to configure the `Accelerator` in addition to distributed launching parameters.
2. `accelerate launch` - Launches a distributed training job using the configuration file generated by `accelerate config`.

The current `AccelerateTrainer` provides functionality similar to `accelerate launch`, in which it will read the generated configuration file and apply it to the `Accelerator`. However, as Ray Train already provides a distributed launching mechanism with its own configurability, we find diminishing value in parsing the configuration file simply for configuring the `Accelerator`. Additionally, it has been a common misconception that the user _must_ use `AccelerateTrainer` in order to use Accelerate. As such, in this proposal we simplify the story by recommending that users configure the `Accelerator` directly in their `TorchTrainer` training code.

#### Accelerate
```python
from accelerate import Accelerator

accelerator = Accelerator(**accelerator_kwargs)
...
```

#### `AccelerateTrainer` (Current)

```bash
! accelerate config
```

```python
from accelerate import Accelerator
from ray.train.accelerate import AccelerateTrainer

def train_func(config):
    accelerator = Accelerator()
    ...

trainer = AccelerateTrainer(
    train_loop_per_worker=train_func,
    accelerate_config=accelerate_config,
    ...
)
trainer.fit()
```

#### `TorchTrainer` (Proposed)

In this proposal, we recommend configuring the `Accelerator` object directly in the training code, exactly as you would if you were not using Ray Train.

```python
from accelerate import Accelerator
from ray.train.torch import TorchTrainer

def train_func(config):
    accelerator = Accelerator(**accelerator_kwargs)
    ...

trainer = TorchTrainer(
    train_loop_per_worker=train_func,
    ...
)
trainer.fit()
```


**Reporting & Checkpointing:**

This is done in the same way as the aforementioned Torch example.

## Compatibility, Deprecation, and Migration Plan

Ray 2.7: 
- Add and expose the public utilities for making training framework code compatible with Ray Train.
- Update documentation and examples to use the `TorchTrainer` API.
- Mark `AccelerateTrainer`, `TransformersTrainer`, and `LightningTrainer` APIs as deprecated.

Ray 2.8:
- Raise error from `AccelerateTrainer`, `TransformersTrainer`, and `LightningTrainer` APIs.

Ray 2.9:
- Remove `AccelerateTrainer`, `TransformersTrainer`, and `LightningTrainer` APIs.

## Test Plan and Acceptance Criteria

We will updates existing integration tests to use the `TorchTrainer` API to validate the functionality and usability of these changes.
