## Summary

We'd like to change how users of Ray AIR (specifically: Train + Tune) configure permanent storage locations.

Current way:
- Users can specify `RunConfig.local_dir` or `tune.run(local_dir)` for storage on the local node
- Users can specify `SyncConfig.upload_dir` for storage on remote nodes

Future way:
- We have a unified setting `RunConfig.storage_path` that can be set either to a local dir or a remote path.
- Setting the storage path to a cloud or NFS URI (e.g., `s3://`, or `file://` that points to a NFS mount). In these cases, data will be first written to a local cache dir on the worker, and then synced to a subdirectory in the storage path designated by `<experiment_name>/<trial_name>/`.
- Setting the storage path to a purely local URI (e.g., `/home/foo/ray_results`). In this mode, there is no persistence once nodes die, nor is there any syncing. The user will have to figure out which node their result is stored on in order to retrieve their result manually. We would only recommend this mode for single node cluster operation generally.

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

@richardliaw, @ericl

### Shepherd of the Proposal (should be a senior committer)

@richardliaw

## Design and Architecture

- We introduce a new `storage_path` argument to `air.RunConfig` and `tune.run` as the main configuration entrypoints
- If the `storage_path` is set to a remote URI, the `local_path` is read from an environment variable `RAY_AIR_CACHE_DIR`
- Backwards compatibility: If a `local_dir` is passed, we set this environment variable
- In downstream components (`TrialRunner`, `Experiment`, `Trial`), we introduce respective arguments: `storage_path` and `experiment_path`

We retain full backwards compatibility with the current API.

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

#### Syncer class

Currently, when a `local_dir` but no `upload_dir` is specified, we synchronize trial data (e.g. checkpoints)
via the object store. This REP states that this path will be deprecated eventually and data will
not be synced instead.

This is subject to discussion. For now, we can keep this functionality as it is functionally orthogonal to
the rest of the proposal.

### Migration plan

1. PR 1/n: [Clean-up path-related properties in developer API classes](https://github.com/ray-project/ray/pull/33370) 
2. PR 2/n: [Add `.path` properties to Result, ResultGrid, and Checkpoint](https://github.com/ray-project/ray/pull/33410)
3. PR 3/n: [Introduce storage_path parameter to public APIs](https://github.com/ray-project/ray/pull/33463)
4. PR 4/n: Move tests to use new path
5. PR 5/n: Change documentation to use new API everywhere

## Test Plan and Acceptance Criteria

PR 3 will introduce new tests, but won't refactor the current tests. Thus, we ensure that our changes are fully backwards compatible.

PR 4 will then move existing unit tests to the new storage path API. This will also introduce a subset of old tests to make sure the legacy API continues to work. 
