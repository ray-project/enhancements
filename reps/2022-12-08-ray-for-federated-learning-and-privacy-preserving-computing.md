Previous issues:
https://github.com/ray-project/ray/issues/25846

## Summary - Ray for federated learning and privacy-preserving computing
### General Motivation
Federated machine learning and privacy-preserving computing are getting wider. They're obviously distributed systems because they're always across parties(companies or ORGs.) To extend Ray ecosystem to federated learning and privacy-preserving computing domain, we should let users be easy to use Ray, for building their own federated learning and privacy-preserving computing applications without any concern on data privacy, task-attacks, illegal invasion, and etc.

In this proposal, we'd like to build a connector layer on Ray, to provide the ability for users use Ray easily to build their such kind of system, with the secure issues addressed totally.

### Key requirements:
- Provide the ability for users to easily build their such kind of application on Ray.
- Setup different Ray clusters for different parties to avoid uncotrollable complex Ray protocols.
- Have a unified and global-viewed programming mode for different parties.
- Data transmition across parties should be in push mode instead of pull mode.
- Tasks should be driven in multi-controller mode. That means tasks should be driven by themselves(who are inside this party) instead of others.

### Should this change be within ray or outside?
Yes.
1. A `ray/python/ray/fed` directory should be added to ray repo as the connector layer, like ray collective, ray dag.
2. `from ray import fed` this kind of code should be supported in Ray core, and then there is no more requirement that needs to change Ray core code yet.

## Stewardship
### Required Reviewers
### Shepherd of the Proposal (should be a senior committer)


## Design and Architecture
The key words in this design are `multiple controller mode`, `restricted data perimeters`.

### User Examples
#### A Simple Example
```python
from ray import fed

@fed.remote
class My:
    def incr(val):
        return self._val += val

@fed.remote(party="BOB")
def mean(val1, val2):
    return np.mean(val1, val2)


fed.init()

my1 = My.party("ALICE").remote()
my2 = My.party("BOB").remote()

val1 = my1.incr.remote(10)
val2 = my1.incr.remote(20)

result = mean.remote(val1, val2)
fed.get(result)

fed.shutdown()
```

From the very simple example code, we create an actor for ALICE and BOB, and then do an aggregation in BOB. Note that, the 2 actors are not in the same Ray cluster.

#### A Federated Learning Example - Diagram
![image](https://user-images.githubusercontent.com/19473861/206469265-e104888e-50c8-4313-aa18-f2caf0928ef1.png)

ALICE and BOB are doing their local training loop and synchronizing the weights every once in a while. In this proposal, it's easy to author the code on the top of Ray. 👇

#### A Federated Learning Example - Code
```python
import numpy as np
import tensorflow as tf
from sklearn.preprocessing import OneHotEncoder
from tensorflow import keras
from ray import fed


@fed.remote
class My:
    def __init__(self):
        self._model = keras.Sequential()
        self._model.compile(optimizer=keras.Adam(), loss=crossentropy)

    def load_data(self, batch_size: int, epochs: int):
        self._dataset = _load_data_from_local()

    def train(self):
        x, y = next(self._dataset)
        with tf.GradientTape() as tape:
            y_pred = self.model(x, training=True)
            loss = self.model.compiled_loss(y, y_pred)
            self._model.optimizer.apply_gradients(zip(
                tape.gradient(loss, trainable_vars),
                self._model.trainable_variables))

    def get_weights(self, party):
        return self._model.get_weights()

    def update_weights(self, party, weights):
        self._model.set_weights(weights)
        return True


@fed.remote
def mean(party, x, y):
    return np.mean([x, y], axis=0)


def main():
    fed.init()
    epochs = 100, batch_size = 4

    my_alice = My.party("alice").remote()
    my_bob = My.party("bob").remote()

    my_alice.load_data.remote(batch_size, epochs)
    my_bob.load_data.remote(batch_size, epochs)

    num_batchs = int(150 / batch_size)
    for epoch in range(epochs):
        # Locally training inner party.
        for step in range(num_batchs):
            my_alice.train.remote()
            my_bob.train.remote()

        # Do weights aggregation and updating.
        w_a = my_alice.get_weights.remote()
        w_b = my_bob.get_weights.remote()
        w_mean = mean.party('alice').remote(w_a, w_b)
        result = fed.get(w_mean)
        print(f'Epoch {epoch} is finished, mean is {result}')
        n_wa = my_alice.update_weights.remote(w_mean)
        n_wb = my_bob.update_weights.remote(w_mean)

    fed.shutdown()
```

### Understanding the mechanism
In this section, we're introducing the deep dive of the above federated learning example, to help understand the mechanism of this connector layer.

Firstly,  ALICE and BOB companies need to create their Ray clusters, and expose one communication port to each other.  
And then, both ALICE and BOB need to run the example code in their Ray clusters(drive the DAG in multi-controller mode). Note that, the peer port and parties info should be passed via `fed.init()`.
![image](https://user-images.githubusercontent.com/19473861/206469580-468ea258-407e-40cf-b059-06a63a25c168.png)

Every Ray cluster will create the Ray tasks specified to its party, meanwhile, it will send the output data  to peer if the downstream Ray task are not specified to its own party. For example, in ALICE cluster, `mean.party('alice').remote(w_a, w_b)` will create a Ray task in ALICE cluster, but it wouldn't be created in BOB cluster. Due to `w_b` is an output data from BOB cluster, therefore, in ALICE cluster, it will be inserted a `recv_op` barrier to `mean` task, and in BOB cluster, it will be inserted a `send_op` barrier as a downstream of `w_b = my_bob.get_weights.remote()` task.

### Benefits 
Compared with running the DAG in one Ray cluster, we have the following significant benefits:
- It's very close to Ray native programming pattern. But the DAG could be ran across Ray clusters.
- Very restricted and clear data perimeters. we only transmit the data which is needed by others. And all of the data couldn't be fetched in any way if we don't allow.
- If we run the DAG in one Ray cluster, data transmition is in PULL-BASED mode. For example, if BOB invokes `ray.get(f.options("PARTY_ALICE").remote())`, the return object is pulled by BOB, as a result, ALICE don't have the knowledge for that. In this proposal, it's in PUSH-BASED mode. ALICE has the knowledge for that there is a data object will be sent to BOB. This is a significant advantage of multi-controller mode.
- All the disadvanteges are addressed in this proposal: code distribution, data privacy, task-attacks, illegal invasion, deserialization vulnerability，and etc.
- Brings all Ray advantages to the local Ray cluster, like highly performance RPC, fault tolerance, task scheduling/resource management, and other Ray ecosystem libraries（ Ray datasets, Ray train and Ray serve） for local training.



## Compatibility, Deprecation, and Migration Plan
None.

## Test Plan and Acceptance Criteria
- Test should includes unit tests and integration tests(release tests).
- Tests should be enabled in CI.
- Benchmark on across parties data transmition for typical federated learning case.
- Document page and necessary code comments.

## (Optional) Follow-on Work
- UX improvements for across parties exceptions passing.
- Provide single-controller mode for quickly building applications and debug inner one party.
- Performance optimization for other workloads.
