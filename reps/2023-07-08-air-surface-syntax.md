# Refining the Ray AIR Surface API

## Summary

Disband the `ray.air` namespace and get rid of Ray AIR sessions.

### General Motivation

Ray AIR has made it significantly easier to use Ray's scalable machine learning
libraries
- Ray Data for batch inference and last mile data processing and ingestion,
- Ray Train for machine learning training and
- Ray Serve for model and application serving
together.

One piece of feedback we have frequently received from users is that they are confused how Ray AIR
relates to the individual libraries. In particular:
- When should I use AIR's abstractions (e.g. should I use `BatchPredictor` or use Ray Data's map functionality,
should I use `PredictorDeployment` or deploy my model with Ray Serve directly?) and
- How does the `ray.air` namespace relate to `ray.data`, `ray.train` and `ray.serve`?
The `ray.air` namespace both containing low level common utilities as well as highler level
abstraction adds to this confusion. We have also learned that the higher level abstractions we
originally introduced for Ray AIR become unneccessary and the same functionality can nicely be achieved
with the libraries themselves by making the libraries a little more interoperable.

We have already implemented this strategy by replacing `BatchPredictor` with Ray Data native functionality
(see https://github.com/ray-project/enhancements/blob/main/reps/2023-03-10-batch-inference.md and
https://docs.ray.io/en/master/data/batch_inference.html) and by
improving Ray Train's ingestion APIs
(https://github.com/ray-project/enhancements/blob/main/reps/2023-03-15-train-api.md and
https://docs.ray.io/en/master/ray-air/check-ingest.html).

As a result of these changes, the `ray.air` namespace has become less and less relevant, and in this
REP we propose to go all the way and remove it altogether in line with the Zen of Python
```
There should be one -- and preferably only one -- obvious way to do it.
```
This solves the confusions mentioned above and makes the Ray AIR APIs more coherent and focused around
the cricital workloads (`ray.data` for batch inference, `ray.train` for training and `ray.serve` for serving).

### Should this change be within `ray` or outside?

main `ray` project. Changes are made to Ray Train, Tune and AIR.

## Stewardship

### Required Reviewers
The proposal will be open to the public, but please suggest a few experienced Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@matthewdeng, @krfricke

### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

@pcmoritz

## Details of the API changes

Concretely, we replace the Ray AIR session with a training context to
1. avoid the user confusion of what a `session` is (and not having to explain in the documentation) and
2. bring the API in line with other Ray APIs like `get_runtime_context` as well as Ray Data's `DataContext`.

The API changes are
```
from ray import air, train

# Ray Train methods and classes:

air.session.report               -> train.report
air.session.get_dataset_shard    -> train.get_dataset_shard
air.Checkpoint                   -> train.Checkpoint
air.Result                       -> train.Result

# Ray Train configurations:

air.config.CheckpointConfig      -> train.CheckpointConfig
air.config.FailureConfig         -> train.FailureConfig
air.config.RunConfig             -> train.RunConfig
air.config.ScalingConfig         -> train.ScalingConfig

# Ray TrainContext methods:

air.session.get_checkpoint       -> train.get_context().get_checkpoint
air.session.get_experiment_name  -> train.get_context().get_experiment_name
air.session.get_trial_name       -> train.get_context().get_trial_name
air.session.get_trial_id         -> train.get_context().get_trial_id
air.session.get_trial_resources  -> train.get_context().get_trial_resources
air.session.get_trial_dir        -> train.get_context().get_trial_dir
air.session.get_world_size       -> train.get_context().get_world_size
air.session.get_world_rank       -> train.get_context().get_world_rank
air.session.get_local_rank       -> train.get_context().get_local_rank
air.session.get_local_world_size -> train.get_context().get_local_world_size
air.session.get_node_rank        -> train.get_context().get_node_rank
```

These changes are ready to try out with https://github.com/ray-project/ray/pull/36706 and we encourage user feedback on the changes.

## Open Questions

We are likely going to remove `PredictorWrapper` and `PredictorDeployment` and migrate the examples to use Ray Serve deployments
direcly, and we are also likely going to move `air.integrations` to `train.integrations` and tentatively the predictors to `ray.train`.

## Migration Plan

We acknowledge that these kinds of API changes are very taxing on our users and we paid special attention that the migration can be done
easily as a simple text substitution without needing large changes for existing code bases. To enable a smooth migration, both APIs will
work for the Ray 2.7 release.

Examples and documentation will be fully converted by Ray 2.7 and the old versions of the APIs will print deprecation warnings together
with instructions on how the user code needs to be upgraded.
