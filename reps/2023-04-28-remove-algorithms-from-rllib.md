## Summary

We'd like to remove aprox ~20 of the 30 algorithms from RLlib. We are considering doing this because 

1. It would reduce the maintenance burden of RLlib.
2. Many of these algorithms have successors that are more performant and/or easier to hyperparameter tune.
3. The proposed algorithms have low ussage by the community according to our ray cluster telemetry readings.

However, we recognize that the value that these algorithms have to the community, and so we are establishing the
`rayproject/rllib-contrib` repo to house these algorithms. Each deprecated algorithm will have its own directory
in this repo containing the following:

1. The implementation of the algorithm.
2. Any tests that are associated with ensuring the correctness of the algorithm.
3. End to end example scripts that demonstrate how to run the algorithm
4. The necessary requirements for running the algorithm
5. boiler plate that allows the algorithm to be installed as a pip package.


## Ussage

Users will be able to replace the deprecated algorithms by doing the following:

1. use git to clone `rayproject/rllib-contrib`
2. navigate to the directory of the previously deprecated algorithm
   for ex. `cd rllib-contrib/maml`
3. run `pip install -e .` to install the algorithm as a pip package
4. Import the algorithm, its policies and its config from that package instead of from `ray.rllib`

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


This proposed algorithms to remove are:
- A3C
- AphaStar
- AlphaZero
- ApexDQN
- ApexDDPG
- DDPG
- ARS
- Bandits 
- CRR 
- DDPG
- DDPPO
- Dreamer
- DT
- ES
- Leelachess
- MADDPG
- MAML
- PG
- QMix
- Random
- SimpleQ
- SlateQ
- TD3
- R2D2


### Shepherd of the Proposal (should be a senior committer)

@sven1977

## Design and Architecture

- We introduce the new  repo for deprecated RLlib algorithms and community contributed algorithms.

### `rayproject/rllib-contrib` File Structure

```
README.md
.github/
    workflow/
        algorithm_foo.yml <---------- This will launch the unit test for algorithm foo with gh actions
algorithm_foo/
    src/
        algorithm_foo.py
    tests/
    examples/
    requirements.txt
    pyproject.toml  <---------- Used for installation

latest_results/
    algorithm_foo/
        results.json
        results.csv
        results.pkl
        results.tensorboard
        results.md  <---------- This will contain some pictures of the performance as a sanity check
```

### Installation
1. use git to clone `rayproject/rllib-contrib`
2. navigate to the directory of the previously deprecated algorithm
   for ex. `cd rllib-contrib/maml`
3. run `pip install -e .` to install the algorithm as a pip package
4. Import the algorithm, its policies and its config from that package instead of from `ray.rllib`


## Contribution Guidlines


The RLlib team commits to the following level of support for the algorithms in this repo:

| Platform | Purpose | Estimated Response Time | Support Level |
| --- | --- | --- | --- |
| [Discuss Forum](https://discuss.ray.io) | For discussions about development and questions about usage. | < 1 week | Community |
| [GitHub Issues](https://github.com/ray-project/rllib-contrib-maml/issues) | For reporting bugs and filing feature requests. | N/A | Community |
| [Slack](https://forms.gle/9TSdDYUgxYs8SA9e8) | For collaborating with other Ray users. | < 2 weeks | Community |

**This means that any issues that are filed will be solved best-effort by the community and there is no expectation of maintenance by the RLlib team.**

We will generally accept contributions to this repo that meet any of the following criteria:
1. Updating dependencies.
2. Submitting community contributed algorithms that have been tested and are ready for use.
3. Enabling algorithms to be run in different environments (ex. adding support for a new type of gym environment).
4. Updating algorithms for use with the newer RLlib API.
5. General bug fixes.

We will not accept contributions that generally add significant new maintenance burden.

### Contributing new algorithms
If you would like to contribute a new algorithm to this repo, please follow the following steps:
1. Create a new directory with the same structure as the other algorithms
2. Add a `README.md` file that describes the algorithm and its usecases
3. Create tests for it that run in under 10 minutes on a single machine (gh actions runners are 2 core machines). Then create a script that can be used as a long running benchmark for your algorithm.
4. Submit a PR and a RLlib maintainer will review it and help you set up your testing to integrate with the CI of this repo.


### Telemetry and Promoting Algorithms to RLlib

In Ray we have telemetry features that allow us to track the usage of algorithms. We'll have to establish a similar
telemetry system for this repo. If an algorithm shows considerable ussage compared to the ussage that we see in RLlib, we can consider promoting it to RLlib and moving the maintenance burden to the RLlib team.


### Testing

Testing will be done with github actions. Each algorithm will have its own github action workflow that will run the unit tests under the algorithm's `tests/` directory. A separate workflow will be created for every algorithm that allows for a long running test to be launched on the Anyscale platform using Anyscale jobs. This test is mainly for performance checking and can only be triggered by a code owner. Testing is only triggered on pull requests. This is because we don't expect the dependencies of the library to change / not be hard-pinned and to reduce the maintenance of the ci.


## Compatibility, Deprecation, and Migration Plan

### Compatibility
The dependencies of each algorithm are hard pinned for each algorithm's package. This means that if a user wants to use a newer version of ray, the burden of migrating the algorithm to the new version of ray is on the user.

### Deprecation
1. We will keep the deprecated algorithms' classes around in the repo for at least 3 releases. We will add standard deprecation warnings to the algorithms and their components, but after the 3 releases we will add deprecation errors
any time an algorithm or its related policy/model components are used.

2. We will add thorough documentation to https://docs.ray.io/en/latest/rllib/rllib-algorithms.html that explains why the algorithms were deprecated and how to migrate to the new rllib contrib algorithm python packages.



## Stewardship

### Required Reviewers

@kouroshhakha, @gjoliver, @richardliaw, @gjoliver, @sven1977, @ArturNiederfahrenhorst



