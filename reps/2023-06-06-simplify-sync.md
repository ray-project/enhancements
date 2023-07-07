# Consolidated persistence API for Ray Train/Tune

## Summary

Standardize on the `storage_path` option as the sole implementation of distributed sync in Ray Train(/Tune). Deprecate the legacy head node syncer code, and add a `storage_filesystem` option for users to customize the underlying pyarrow filesystem used for `storage_path` sync.

Relatedly, simplify the Train `Checkpoint` API to also standardize on `pyarrow.fs` as its backing implementation.

### General Motivation

The motivation for this change is two-fold.

First, it cuts down on the number of alternative options and implementations users have for persistent storage in Ray Train and Tune. This materially reduces the docs and examples (i.e., going from multiple pages of persistence-related docs to one page).

Second, it reduces the implementation and API surface that the ML team has to maintain. Many of these storage abstractions were introduced in the initial years when Ray was created. At the time, filesystem abstractions for Python were less prevalent. Today, `pyarrow.fs` (in conjunction with fsspec) are industry standards for accessing remote storage, and Ray already has a hard dependency on pyarrow in Ray Data.

Both of these have been significant pain points over the past year, based both on user and internal feedback. This shouldn't be too surprising, as the persistent storage APIs have never been significantly revised before, beyond introduction of the new `storage_path` option, and similarly the Checkpoint API is in beta.

### Should this change be within `ray` or outside?

main `ray` project. Changes are made to Ray Train and Tune.

## Stewardship

### Required Reviewers
The proposal will be open to the public, but please suggest a few experienced Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@matthewdeng, @krfricke

### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

@pcmoritz

## User Story

Persistence will center around the recently introduced `storage_path` option.

### Persistent trial directory

Ray Train provides the abstraction of a persistent trial directory. Each trial has its current working directory set to a trial specific directory (e.g., `~/ray_results/experiment_1/trial_1`). Train will synchronize this trial directory with persistent storage specified via the `storage_path` option.

This directory is persistent since Train is syncing it to persistent storage for the user. The experiment driver will record experiment results and other metadata to a similar directory on the head node, which will also synchronize to persistent storage.

![overview](user-story.png)

### Checkpoints

While trials can write arbitrary data to the persistent trial directory, they cannot be guaranteed they can read it back on checkpoint/restore. Data that is needed for resume must be recorded via the `train.report(checkpoint)` API.

For example, this is how you could record a Torch checkpoint:

```python
   torch.save(model, "path/to/dir")
   train.report(checkpoint=Checkpoint.from_directory("path/to/dir"))
```

Train will make a copy of the specified checkpoint data in the trial dir for persistence. The checkpoint data is managed by Train (e.g., Train may restore only the latest checkpoint or delete previous checkpoints to save space according to checkpoint policy).

The files of the recorded checkpoint can be accessed via `result.checkpoint`. The `Checkpoint` object itself is a logical tuple of `(path, pyarrow.fs.FileSystem)`. Users can also get and set arbitrary metadata to these checkpoints (e.g., preprocessor configs, model settings, etc), which will be recorded in a `metadata.json` file.

Checkpoints can be recorded from multiple ranks. By default, only checkpoint data from rank zero is preserved. Data from all ranks can be retained via a `keep_all_ranks` option.

### FAQ

- What's in this persistent trial directory?

    This directory contains metric files written by Train loggers, such as JSON metrics, checkpoint data managed by Train, as well as arbitrary artifacts created by a trial.

- What guarantees does Train make about persistence?

    Train will sync this directory to persistent storage according to the SyncConfig settings for the run (e.g., once every 10 min or on checkpoint).

- What can `storage_path` be set to?

	Storage path can be set to a NFS mount in a cluster, a local path on a single-node cluster, or a cloud storage URI supported by pyarrow. In these cases, Train will use the `pyarrow.fs.sync_files` API to copy data to the specified storage path.

    If storage path is not set, sync is disabled, and an error is raised if a remote trial tries to persist a checkpoint.

- What if I do not have NFS or cloud storage?

	Train no longer supports persistence for distributed experiments unless you specify a `storage_path`. You can provide a custom Arrow filesystem to support custom storage solutions.

## Open Questions

- For multi-rank checkpointing, should we consolidate outputs from multiple ranks into a single directory, retain the outputs in separate directories (e.g., `rank_{i}`), or provide options to support both?

- For backwards compatibility, should we provide a (slower, limited) head-node remote filesystem implementation?

## Rollout Plan

### Impact of Changes

This is a breaking change for users that do not already use NFS or cloud storage. For other users, the changes are for the most part limited to beta APIs (i.e., Checkpoint).

We are evaluating the impact of deprecating head node storage, by rolling the breaking change out early in Ray 2.6. This is to inform whether we should provide extended backwards compatibility.

### Timeline

- Ray 2.6: Head node sync is disabled and will raise an exception. It is possible to re-enable by setting a flag.
- Ray 2.7: Head node sync will be fully removed.

## Examples:

Please refer to the prototype PR: https://github.com/ray-project/ray/pull/36969

## Test Plan and Acceptance Criteria

The new persistence behavior will be fully covered by integration tests. We plan to remove the old tests in Ray 2.7 that cover legacy behavior.
