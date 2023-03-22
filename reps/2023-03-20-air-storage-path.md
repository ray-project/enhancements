## Summary

We'd like to change how users of Ray AIR (specifically: Train + Tune) configure permanent storage locations.

Current way:
- Users can specify `RunConfig.local_dir` or `tune.run(local_dir)` for storage on the local node
- Users can specify `SyncConfig.upload_dir` for storage on remote nodes

Future way:
- All remote storage is standardized on the built in RAY_STORAGE mechanism.
- We have a unified setting `RunConfig.storage_path` that can be used to override the default RAY_STORAGE path configured for the cluster.
- Setting the storage path to a cloud or NFS URI (e.g., `s3://`, or `file://` that points to a NFS mount). In these cases, data will be first written to a local cache dir on the worker, and then synced to a subdirectory in the storage path designated by `<experiment_name>/<trial_name>/`.
- Setting the storage path to a purely local URI (e.g., `/home/foo/ray_results`). In this mode, trial data is synced to the head node via the object store. We would generally recommend using a remote storage path or shared directory instead.


```python
import ray
from ray import air, tune

results = tune.Tuner(train_fn, run_config=air.RunConfig(storage_path="s3://foo/bar")).fit()
assert results.best_checkpoint().path.startswith("s3://foo/bar")
```

### Background: Persistent storage in Ray Tune

Running a Ray Tune run creates a number of artifacts:

1. Experiment-level data, such as the search configuration
2. Trial-level, driver-based data, such as 
   - The configuration the trial used
   - The metrics (results) the trial reported (as CSV, JSON, TensorBoard)
   - Any errors the trial logged
3. Trial-level, trainable-based data, such as
   - Checkpoints
   - Local logfiles (e.g. stdout/stderr log of the trainable)
   - Other user-created artifacts

1 and 2 are handled by the driver. 3 is handled by the trainable (i.e. a remote actor).

This data is stored as follows:

- Assume we have a `storage_path=/path/to/storage` (can be local, shared, remote)
- Then, the experiment-level data will be stored in `/path/to/storage/experiment_name`
- The trial-based data will be stored in `/path/to/storage/experiment_name/trial_id`
- This is true for both the driver-based trial data and the trainable-based trial data

If the respective trial is running on the head node, this means that both the
driver-based and the trainable-based trial data will be saved in the same directory.

The same is true if a shared filesystem (e.g. NFS) is configured.

If the trial is running on a worker node, the trial directories will have only partial contents
on different nodes: The head node will have the driver-based data, and the worker node
will have the trainable-based data.

In this case, we need to synchronize the data to a common location, so that we have the full
data in one place.

Specifying this common persistent storage location and discussing the implementation of
the synchronization is the subject of this REP.

### General Motivation

Users care about where their data is ultimately stored. They usually don't care about
how it got there.

However, currently they specify a local storage directory, from which data is periodically "synced"
to a remote storage location. This flow (collect locally, sync to storage) is more of an implementation
detail.

Instead, users should just specify where their data ends up - with `storage_path`.

In the long run, this concept helps us simplify the user journey and removes confusion
in the configuration.

Risk: Since we will continue to use locally cached data for the implementation,
hiding the location in e.g. an environment variable can make failures more opaque
to users.


### Should this change be within `ray` or outside?

This is an API change to Ray AIR (specifically, `air.RunConfig`, `tune.SyncConfig`, and `tune.run()`)
and therefore has to be within Ray.


## Stewardship

### Required Reviewers

@richardliaw, @ericl, @gjoliver

### Shepherd of the Proposal (should be a senior committer)

@richardliaw

## Design and Architecture

- We introduce a new `storage_path` argument to `air.RunConfig` and `tune.run` as the main configuration entrypoints
- The `storage_path` argument defaults to the configured `RAY_STORAGE` environment variable
- If the `storage_path` is set to a remote URI, the `local_path` is read from an environment variable `RAY_AIR_CACHE_DIR`
- Backwards compatibility: If a `local_dir` is passed, we set this environment variable
- In downstream components (`TrialRunner`, `Experiment`, `Trial`), we introduce respective arguments: `storage_path` and `experiment_path`

We retain full backwards compatibility with the current API.

## Synchronization

The synchronization to a common location already works today. However, we can improve this
for better efficiency and to better define the responsibilities of different components.

### Current implementation

The current flow multiplexes between two scenarios: With cloud storage enabled or without it.

When cloud storage is enabled:

- The driver saves experiment data to `/local/path/experiment_name`
- The driver saves driver-based trial data to `/local/path/experiment_name/trial_id`
- The driver uses a `Syncer` to periodically sync this state to the remote storage (e.g. S3), e.g. `s3://bucket/location/experiment_name`
- This _includes_ driver-based trial data, such as the config and results log
- This _excludes_ trainable-based trial data, such as checkpoints.
  Uploading checkpoints is the trainable's responsibility.
- The trainable saves trial-based data such as checkpoints to their local `/local/path/experiment_name/trial_id`
- The trainable uses a `Syncer` to upload these checkpoints _synchronously_ to the remote storage after saving, e.g. `s3://bucket/location/experiment_name/trial_id/checkpoint_001`

When cloud storage is disabled:

- The driver saves experiment data to `/local/path/experiment_name`
- The driver saves driver-based trial data to `/local/path/experiment_name/trial_id`
- The trainable saves trial-based data such as checkpoints to their local `/local/path/experiment_name/trial_id`
- The driver uses a `SyncerCallback` to synchronize contents form the trial directories (i.e. the missing trial-based data).
  This uses the Ray object store.
- This also happens _synchronously_ every time a trial reports a checkpoint has been saved

Currently, the object store-based transfer serializes all the data at the same time. Thus, the heap
memory must be large enough to hold all the data to be transferred between nodes.

### Future implementation

In the first step (when we just change the API), this flow can remain the same, as it fulfills
all requirements.

The implementation for the case when a remote storage location is defined will remain as is.

However, the implementation for the local storage path is currently inefficient because 
the driver synchronously blocks until trial-based data is synchronized using the `SyncerCallback`.

Instead, we should move this synchronization into the trial - analogous to the remote storage case.

Thus, when a local storage path is defined (cloud storage is disabled):

- The driver saves experiment data to `/local/path/experiment_name`
- The driver saves driver-based trial data to `/local/path/experiment_name/trial_id`
- The trainable saves trial-based data such as checkpoints to their local `/local/path/experiment_name/trial_id`
- **The trainable uses the Ray object store to transfer these checkpoints _synchronously_ to the head node.**
- Technically, the trainable will schedule an actor on the head node to receive the data.
  (this may be a detached named actor to avoid scheduling too many processes at the head node).

The trainable can detect if the trial shares the storage path with the head node
(e.g. if RAY_STORAGE is set, or if it's running on the same node). This then 
removes the need for any syncing to happen. This will also cover the case when
shared storage (e.g. NFS) is defined.

## Usage Example

### Basic usage

```python
import ray
from ray import air, tune

results = tune.Tuner(train_fn, run_config=air.RunConfig(storage_path="s3://foo/bar")).fit()
assert results.best_checkpoint().path.startswith("s3://foo/bar")
```

### Usage when Ray storage is configured

```python
import ray
from ray import air, tune

# STORAGE_PATH is configured to s3://foo/bar

results = tune.Tuner(train_fn).fit()
assert results.best_checkpoint().path.startswith("s3://foo/bar")
```


### Usage with local directories

```python
import ray
from ray import air, tune

results = tune.Tuner(train_fn, run_config=air.RunConfig(storage_path="/local/dir")).fit()

# even if train_fn ran on a different node, its logs + artifacts will be in /local/dir
assert results.best_checkpoint().path.startswith("/local/dir")
```

### Accessing storage paths

```python
import ray
from ray import tune

results = tune.Tuner(train_fn).fit()

assert results.path.startswith("s3://bucket/dir/experiment_dir")

result = results.get_best_result()
assert result.path.startswith("s3://bucket/dir/experiment_dir/trial0")

checkpoint = result.checkpoint
assert checkpoint.path.startswith("s3://bucket/dir/experiment_dir/trial0/checkpoint_0010")
```

## Compatibility, Deprecation, and Migration Plan

### Compatibility

We retain full backwards compatibility with the current API.

The initial PR merges will keep the old API in the unit tests.

### Deprecation plan

We are changing stable public APIs. Thus, we will follow a deprecation plan over three releases:

For public APIs (`tune.Tuner()`, `tune.run()`):

- In Ray 2.4, we soft deprecate `RunConfig.local_dir` and `SyncConfig.upload_dir` and warn if they are used
- We keep this behavior in Ray 2.5
- In Ray 2.6 we will raise an error if those arguments are still used
- In Ray 2.7 we remove the old code paths

For developer APIs (`Trial`, `TrialRunner`, `Experiment`) we can speed up hard deprecation
by one release.

### Migration plan

1. PR 1/n: [Clean-up path-related properties in developer API classes](https://github.com/ray-project/ray/pull/33370) 
2. PR 2/n: [Add `.path` properties to Result, ResultGrid, and Checkpoint](https://github.com/ray-project/ray/pull/33410)
3. PR 3/n: [Introduce storage_path parameter to public APIs](https://github.com/ray-project/ray/pull/33463)
4. PR 4/n: Move tests to use new path
5. PR 5/n: Change documentation to use new API everywhere

## Test Plan and Acceptance Criteria

PR 3 will introduce new tests, but won't refactor the current tests. Thus, we ensure that our changes are fully backwards compatible.

PR 4 will then move existing unit tests to the new storage path API. This will also introduce a subset of old tests to make sure the legacy API continues to work. 
