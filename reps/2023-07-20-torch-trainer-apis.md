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

In the following sections, we show how this change will be reflected each of the individual frameworks by comparing:
1. A (minimal) training script for the framework.
2. The training script rewritten using the current Ray Train APIs.
3. The training script rewritten using the proposed Ray Train APIs.

### Lightning

**Lightning:**
```python 
import lightning

class MyLightningModule(lightning.LightningModule):
    ...

trainer = lightning.Trainer(**trainer_kwargs)
trainer.fit(MyLightningModule(**module_kwargs), **fit_kwargs)
```

**`LightningTrainer` (Current):**

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

**`TorchTrainer` (Proposed):**

In this proposal, we provide a few common utilties that the user can use directly in their training code to configure the `Trainer` object.

- `get_devices` - Returns a list of devices to use for training.
- `prepare_trainer` - Validates that the `Trainer` object is configured correctly to be compatible with Ray Train.
- `RayDDPStrategy`, `RayFSDPStrategy`, `RayDeepSpeedStrategy` - `LightningStrategy`s for different training strategies that are compatible with Ray Train.
- `RayEnvironment` - A `LightningEnvironment` that is compatible with Ray Train.
- `RayModelCheckpoint` - A `LightningCallback` that configures the `Trainer` to report metrics and checkpoints to Ray Train.

With this, the user can directly interact with the Lightning interface, and is able define their training logic to use additional functionality such as `lightning.Trainer.test()`.

```python
import lightning
from ray.train.torch import TorchTrainer
from ray.train.lightning import (
    get_devices, 
    prepare_trainer,
    RayDDPStrategy, 
    RayEnvironment, 
    RayModelCheckpoint
)

def train_func(config):

    ...

    devices = get_devices()
    strategy = RayDDPStrategy()
    environment = RayEnvironment()
    checkpoint_callback = RayModelCheckpoint()

    lightning_trainer = lightning.Trainer(
        devices=devices,
        strategy=strategy,
        plugins=[environment],
        callbacks=[checkpoint_callback],
        **trainer_kwargs
    )
    ray.train.lightning.prepare_trainer(lightning_trainer)
    lightning_trainer.fit(MyLightningModule(**module_kwargs), **fit_kwargs)


trainer = TorchTrainer(
    train_loop_per_worker=train_func,
    ...
)
trainer.fit()
```

### HuggingFace Transformers

**Transformers:**
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

**`TorchTrainer` (Proposed):**

In this proposal, we provide a `prepare_trainer` utility that the user can use directly in their training loop code to validate that the `Trainer` object is configured correctly to be compatible with Ray Train.

With this, the user is able define their training logic to use additional functionality such as `transformers.Trainer.evaluate()`.

```python
import transformers
from ray.train.torch import TorchTrainer
from ray.train.huggingface.transformers import prepare_trainer

def train_func(config):
    ...
    transformers_trainer = transformers.Trainer(**trainer_kwargs)
    prepare_trainer(transformers_trainer)
    trainer.train()

trainer = TorchTrainer(
    train_loop_per_worker=train_func,
    ...
)
trainer.fit()
```

### HuggingFace Accelerate

At its core, HuggingFace Accelerate can be used by configuring and instantiating an `Accelerator` object in your training code.

HuggingFace Accelerate also provides two **optional** CLI commands:
1. `accelerate config` - Generates a configuration file that is used to configure the `Accelerator` in addition to distributed launching parameters.
2. `accelerate launch` - Launches a distributed training job using the configuration file generated by `accelerate config`.

The current `AccelerateTrainer` provides functionality similar to `accelerate launch`, in which it will read the generated configuration file and apply it to the `Accelerator`. However, as Ray Train already provides a distributed launching mechanism with its own configurability, we find diminishing value in parsing the configuration file simply for configuring the `Accelerator`. Additionally, it has been a common misconception that the user _must_ use `AccelerateTrainer` in order to use Accelerate. As such, in this proposal we simplify the story by recommending that users configure the `Accelerator` directly in their `TorchTrainer` training code.

**Accelerate:**
```python
from accelerate import Accelerator

accelerator = Accelerator(**accelerator_kwargs)
...
```

**`AccelerateTrainer` (Current):**

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

**`TorchTrainer` (Proposed):**

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

## Compatibility, Deprecation, and Migration Plan

Ray 2.7: 
- Add and expose the public utilities for making training framework code compatible with Ray Train.
- Update documentation and examples to use the `TorchTrainer` API.
- Mark `AccelerateTrainer`, `TransformersTrainer`, and `LightningTrainer` APIs as deprecated.

Ray 2.8:
- Remove `AccelerateTrainer`, `TransformersTrainer`, and `LightningTrainer` APIs.

## Test Plan and Acceptance Criteria

We will updates existing integration tests to use the `TorchTrainer` API to validate the functionality and usability of these changes.
