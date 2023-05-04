## Summary

We'd like to create the `RLlib-contrib` repository for community contributed algorithms and algorithms with low usage in RLlib. We'd like to start by migrating approximately ~25 of the 30 algorithms from RLlib into `RLlib-contrib`. We are considering doing this because 

1. Doing so will greatly increase the ability / lower the barrier of entry of community members to contribute new algorithms to RLlib.
2. It would reduce the maintenance burden of RLlib.
3. Many of these algorithms [to be migrated to this repo] have successors that are more performant and/or easier to hyperparameter tune.
4. The proposed algorithms have low usage by the community according to our ray cluster telemetry readings.

However, as we do recognize the value that these algorithms have to the community, we are establishing the
`rayproject/rllib-contrib` repo to house these algorithms. Each deprecated algorithm will have its own directory
In this repo containing the following:

1. The implementation of the algorithm.
2. Any tests that are associated with ensuring the correctness of the algorithm.
3. End to end example scripts that demonstrate how to run the algorithm.
4. The necessary requirements for running the algorithm (for example ray at a specific version).
5. Boiler plate that allows the algorithm to be installed as a pip package.


## Usage

Users will be able to replace the deprecated algorithms by doing the following:

In most usecases, the migration will look like replacing imports inside of an experiment script.

For example:

```python
from gymnasium.wrappers import TimeLimit

import ray
from ray import air
from ray import tune
from ray.rllib.examples.env.cartpole_mass import CartPoleMassEnv
from ray.tune.registry import register_env

# from ray.rllib.algorithms.maml import MAML, MAMLConfig  # BEFORE
from rllib_maml.maml import MAML, MAMLConfig  # AFTER

if __name__ == "__main__":
    ray.init()
    register_env(
        "cartpole",
        lambda env_cfg: TimeLimit(CartPoleMassEnv(), max_episode_steps=200),
    )

    config = MAMLConfig()

    tuner = tune.Tuner(
        MAML,
        param_space=config.to_dict(),
        run_config=air.RunConfig(stop={"training_iteration": 100}),
    )
    results = tuner.fit()

```


The proposed algorithms to migrate from RLlib are outlined in the table below, along with the estimated:

| Algorithm | Soft Deprecation Release # | Hard Deprecation Release # |
| --- | --- | --- |
| A3C | 2.5 | 2.8 |
| A2C | 2.5 | 2.8 |
| R2D2 | 2.5 | 2.8 |
| MAML | 2.5 | 2.8 |
| AphaStar | 2.6 | 2.9 |
| AlphaZero | 2.6 | 2.9 |
| ApexDQN | 2.6 | 2.9 |
| ApexDDPG | 2.6 | 2.9 |
| DDPG | 2.6 | 2.9 |
| ARS | 2.6 | 2.9 |
| Bandits | 2.6 | 2.9 |
| CRR | 2.6 | 2.9 |
| DDPG | 2.6 | 2.9 |
| DDPPO | 2.6 | 2.9 |
| Dreamer | 2.6 | 2.9 |
| DT | 2.6 | 2.9 |
| ES | 2.6 | 2.9 |
| Leelachess | 2.6 | 2.9 |
| MADDPG | 2.6 | 2.9 |
| MBMPO | 2.6 | 2.9 |
| PG | 2.6 | 2.9 |
| QMix | 2.6 | 2.9 |
| Random | 2.6 | 2.9 |
| SimpleQ | 2.6 | 2.9 |
| SlateQ | 2.6 | 2.9 |
| TD3 | 2.6 | 2.9 |


## Design and Architecture

- We introduce the new repo for deprecated RLlib algorithms and community contributed algorithms.

### `rayproject/rllib-contrib` File Structure

```
README.md
.github/
    workflow/
        algorithm_foo.yml <---------- This will launch the unit test for algorithm foo with gh actions
algorithm_foo/
    src/
        algorithm_foo/
            __init__.py
            algorithm_foo.py
    tests/
    examples/
    latest_results/
        results.json
        results.csv
        results.pkl
        results.tensorboard
        results.md  <---------- This will contain some pictures of the performance as a sanity check
    requirements.txt
    pyproject.toml  <---------- Used for installation
    
```

### Installation

Either install from pypi:

` pip install rllib-contrib-ALGORITHM`

or install from source:

1. Use git to clone `rayproject/rllib-contrib`.
2. Navigate to the directory of the previously deprecated algorithm
   for ex. `cd rllib-contrib/maml`.
3. Run `pip install -e .` to install the algorithm python module as a pip package.
4. Import the algorithm, its policies and its config from that package instead of from `ray.rllib`.


## Contribution Guidlines


The RLlib team commits to the following level of support for the algorithms in this repo:

| Platform | Purpose | Support Level |
| --- | --- | --- |
| [Discuss Forum](https://discuss.ray.io) | For discussions about development and questions about usage. | Community |
| [GitHub Issues](https://github.com/ray-project/rllib-contrib-maml/issues) | For reporting bugs and filing feature requests. | Community |
| [Slack](https://forms.gle/9TSdDYUgxYs8SA9e8) | For collaborating with other Ray users. | Community |

**This means that any issues that are filed will be solved best-effort by the community and there is no expectation of maintenance by the RLlib team.**

We will generally accept contributions to this repo that meet any of the following criteria:
1. Updating dependencies.
2. Submitting community contributed algorithms that have been tested and are ready for use.
3. Enabling algorithms to be run in different environments (ex. adding support for a new type of gym environment).
4. Updating algorithms for use with the newer RLlib APIs.
5. General bug fixes.

We will not accept contributions that generally add significant new maintenance burden. In this case users should instead make their own repo with their contribution, **using the same guidelines as this repo** and the RLlib team can help to market/promote it in the ray docs.

### Contributing new algorithms

If you would like to contribute a new algorithm to this repo, please follow the following steps:
1. Create a new directory with the same structure as the other algorithms.
2. Add a `README.md` file that describes the algorithm and its usecases.
3. Create unit tests/shorter learning tests and long learning tests for the algorithm.
4. Submit a PR and a RLlib maintainer will review it and help you set up your testing to integrate with the CI of this repo.

Regarding unit tests and long running tests:

- Unit tests are any tests that tests a sub component of an algorithm. For example tests that check the value of a loss function given some inputs.
- Short learning tests should run an algorithm on an easy to learn environment for a short amount of time (e.g. ~3 minutes) and check that the algorithm is achieving some learning threshold (e.g. reward mean or loss).
- Long learning tests should run an algorithm on a hard to learn environment (e.g.) for a long amount of time (e.g. ~1 hour) and check that the algorithm is achieving some learning threshold (e.g. reward mean or loss).


### Telemetry and Promoting Algorithms to RLlib

In Ray we have telemetry features that allow us to track the usage of algorithms. We'll have to establish a similar telemetry system for this repo. If an algorithm shows considerable usage compared to the usage that we see in RLlib, we can consider promoting it to RLlib and moving the maintenance burden to the RLlib team. 

It isn't crucial to add this telemetry system in for the initial release of this repo, but it is something that we will add in the future. This is because we don't expect the usage of this repo to be high initially.

### Testing

Testing will be done with github actions. Each algorithm will have its own github action workflow that will run the unit tests under the algorithm's `tests/` directory. A separate workflow will be created for every algorithm that allows for a long running test to be launched on the Anyscale platform using Anyscale jobs. This test is mainly for performance checking and can only be triggered by a code owner. Testing is only triggered on pull requests. This is because we don't expect the dependencies of the library to change / not be hard-pinned and to reduce the maintenance of the CI. Finally unit tests for
an algorithm will only be launched if that algorithm has been modified or added. This is to reduce the
amount of time that the CI takes to run.


## Compatibility, Deprecation, and Migration Plan

### Compatibility
The dependencies of each algorithm are hard pinned for each algorithm's package. This means that if a user wants to use a newer version of ray, the burden of migrating the algorithm to the new version of ray is on the user.

### Deprecation
1. We will keep the deprecated algorithms' classes around in the repo for at least 3 releases. We will add standard deprecation warnings to the algorithms and their components, but after the 3 releases we will add deprecation errors any time an algorithm or its related policy/model components are used.

2. We will add thorough documentation to https://docs.ray.io/en/latest/rllib/rllib-algorithms.html that explains why the algorithms were deprecated and how to migrate to the new rllib contrib algorithm python packages.


## Stewardship

### Shepherd of the Proposal

@sven1977

### Required Reviewers

@kouroshhakha, @gjoliver, @richardliaw, @gjoliver, @sven1977, @ArturNiederfahrenhorst



