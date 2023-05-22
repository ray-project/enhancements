## Summary
### General Motivation
**Core Motivation**:
1. Improve the performance of batch calling ActorTask. The current performance bottleneck of batch invoking ActorTask mainly lies in the duplicate serialization of parameters, repeated puts parameters into object store, and frequent context switching between Python and C++. The Batch remote API will optimize performance for these three aspects.


In distributed computing scenarios, such as big data computing、AI training and inference, there is a need to send a large number of RPC requests(ActorTask) to multiple Actors in batches.  
For example, in a typical star-topology architecture with one master Actor and 400 worker Actors, the same computational request needs to be sent to 400 worker Actors. 
However, the computing tasks of each Worker Actor are very short. In such a scenario, the performance requirements for executing batch actor task remote of a large number of Actor are very high.  
Therefore, for the scenario of batch calling Actor tasks, I want to add a new optimization API, batch_remote(), to improve the performance of batch submission of Actor Task calls.   
After my own performance testing and comparison, this API interface can improve performance by 2 ~ 100+ times.

Current situation of batch calling actor tasks:
```
actors = [WorkerActor.remote() for _ in range(400)]

# This loop's repeated invocation actually wastes a lot of performance.
for actor in actors:
  actor.compute.remote(args)
```

Using the new Batch Remote API:
```
actors = [WorkerActor.remote() for _ in range(400)]

# Calling it only once can greatly improve performance.
batch_remote_handle = ray.experimental.batch_remote(actors)
batch_remote_handle.compute.remote(args)
```

The current performance bottleneck of batch invoking ActorTask mainly lies in the duplicate serialization of parameters, repeated puts parameters into object store, and frequent context switching between Python and C++.  
The Batch Remote API can reduce the following performance costs(The N is the number of Actors):  
1. Reduce (N-1) times of parameter serialization performance costs.
2. Reduce (N-1) times of putting parameter into object store performance costs for scenarios with large parameters.
3. Reduce (N-1) times of context switching between Python and C++ and repeated parameter verification performance costs.


### Should this change be within `ray` or outside?
This requires adding a new interface in Ray Core.  
The initial consideration is to add it to the ray.experimental module.

## Stewardship
### Required Reviewers
...

### Shepherd of the Proposal (should be a senior committer)
...

## Design and Architecture
### API
Use case  
Plan 1
```
batch_remote_handle = ray.experimental.batch_remote(actors)
batch_remote_handle.compute.remote(args)
```

Plan 2
```
batch_remote_handle = ray.experimental.BatchRemoteHandle(actors)
batch_remote_handle.compute.remote(args)
```

### Implementation

1. In the ActorTask calling logic, the parameter verification, parameter serialization, and parameter putting into the object store can be reused from the original logic.
2. Then, add a BatchSubmitActorTask interface in the C++ CoreWorker layer, and loop through the original SubmitActorTask interface to submit multiple ActorTasks. This way, the batch submission of ActorTasks can be handled in the C++ CoreWorker layer, saving time on parameter serialization and putting into the object store. 

Overall, the implementation is relatively simple, with most of it reusing the original ActorTask submission logic.

```
def batch_remote(actors: List[ray.actor.ActorHandle]) -> BatchRemoteHandle:
  ...

class BatchRemoteHandle:
  def __init__(self, actors: List[ray.actor.ActorHandle]):
    ...
  
  def _actor_method_call():
    object_refs = worker.core_worker.batch_submit_actor_task(actor_ids, ...)
    return object_refs

C++: 
class CoreWorker:
  BatchSubmitActorTask(actor_ids, &return_ids) {
    for( auto actor_id : actor_ids) {
      SubmitActorTask(actor_id, ..., return_ids)
    }
  }
```

### Performance improvement comparison:
We have already implemented and conducted extensive performance testing internally.  
The following are the performance comparison results.

**Table 1: Comparison of remote call time with varying parameter sizes and 400 Actors**


Parameter Size (byte) | Time taken for foreach_remote(ms) | Time taken for batch_remote(ms) | The ratio of time reduction
-- | -- | -- | --
10 | 40.532 | 9.226 | 77.2%
409846 | 584.345 | 24.106 | 95.9%
819446 | 606.725 | 16.019 | 97.4%
1229047 | 976.974 | 19.403 | 98.0%
1638647 | 993.454 | 23.184 | 97.7%
2048247 | 972.438 | 19.028 | 98.0%
2457850 | 987.04 | 17.642 | 98.2%
2867450 | 976.165 | 15.07 | 98.5%
3277050 | 1108.331 | 18.272 | 98.4%
3686650 | 1186.371 | 16.011 | 98.7%
4096250 | 1335.575 | 15.951 | 98.8%
4505850 | 1490.914 | 20.928 | 98.6%
4915450 | 1511.744 | 23.041 | 98.5%
5325050 | 1716.752 | 17.515 | 99.0%
5734650 | 2009.711 | 22.891 | 98.9%
6144250 | 2424.166 | 23.129 | 99.0%
6553850 | 2354.033 | 20.271 | 99.1%
6963450 | 2599.015 | 24.347 | 99.1%
7373050 | 2610.843 | 17.91 | 99.3%
7782650 | 2751.179 | 17.258 | 99.4%

![Comparison of remote call time with varying parameter sizes and 400 Actors](https://github.com/ray-project/ray/assets/11072802/87591859-d247-4444-95ee-e37f04efc095)


**Conclusion:**  
1. The larger the parameter size, the greater the performance gain of batch_remote(). 


**Table 2: Comparison of remote call time with varying numbers of Actors and a fixed parameter size (1MB)**

actor counts | Time taken for foreach_remote(ms) | Time taken for batch_remote(ms) | The ratio of time reduction
-- | -- | -- | --
50 | 95.889 | 4.657 | 95.1%
100 | 196.184 | 8.447 | 95.7%
150 | 291.879 | 15.228 | 94.8%
200 | 373.806 | 21.161 | 94.3%
250 | 475.482 | 20.768 | 95.6%
300 | 577.406 | 24.323 | 95.8%
350 | 646.415 | 24.59 | 96.2%
400 | 763.94 | 34.806 | 95.4%
450 | 874.601 | 30.723 | 96.5%
500 | 955.915 | 45.888 | 95.2%
550 | 1041.728 | 39.736 | 96.2%
600 | 1124.786 | 45.122 | 96.0%
650 | 1262.363 | 46.687 | 96.3%
700 | 1331.427 | 60.543 | 95.5%
750 | 1407.485 | 47.386 | 96.6%
800 | 1555.571 | 55.297 | 96.4%
850 | 1549.03 | 60.493 | 96.1%
900 | 1675.685 | 58.268 | 96.5%
950 | 2314.186 | 48.785 | 97.9%

![Comparison of remote call time with varying numbers of Actors and 1MB parameter size](https://github.com/ray-project/ray/assets/11072802/fe41eb01-24bd-4418-90c1-81b6f413e548)

**Conclusion:**  
The more actors, the greater the performance gain. 


**Table 3: Comparison of remote call time with varying numbers of Actors and no parameters in remote calls**

This test is to confirm the degree of performance optimization after reducing the frequency of switching between the Python and C++ execution layers. 

actor counts | Time taken for foreach_remote(ms) | Time taken for batch_remote(ms) | The ratio of time reduction
-- | -- | -- | --
50 | 2.083 | 1.257 | 39.7%
100 | 4.005 | 2.314 | 42.2%
150 | 5.582 | 3.467 | 37.9%
200 | 8.104 | 3.574 | 55.9%
250 | 10.104 | 4.64 | 54.1%
300 | 11.858 | 6.224 | 47.5%
350 | 13.826 | 8.017 | 42.0%
400 | 15.862 | 8.145 | 48.7%
450 | 18.368 | 9.261 | 49.6%
500 | 18.881 | 10.722 | 43.2%
550 | 21.129 | 11.944 | 43.5%
600 | 23.413 | 12.925 | 44.8%
650 | 26.485 | 13.328 | 49.7%
700 | 27.855 | 14.303 | 48.7%
750 | 29.432 | 14.922 | 49.3%
800 | 31.03 | 16.329 | 47.4%
850 | 32.405 | 17.582 | 45.7%
900 | 34.388 | 18.521 | 46.1%
950 | 36.499 | 19.658 | 46.1%

![Comparison of remote call time with varying numbers of Actors and no parameters in remote calls](https://github.com/ray-project/ray/assets/11072802/493d036b-8eb2-4397-ac8b-492bc5b526b9)


**Conclusion:**  
After comparison, in the scenario of remote calls without parameters, the performance is optimized by 2+ times.

**Table 4: Comparison of remote call time with varying numbers of Actors and object ref parameters in remote calls**

actor counts | The time taken for foreach_remote(ms) | The time taken for batch_remote(ms) | The ratio of time reduction
-- | -- | -- | --
50 | 3.878 | 1.488 | 61.6%
100 | 8.383 | 2.405 | 71.3%
150 | 12.16 | 3.255 | 73.2%
200 | 16.835 | 4.913 | 70.8%
250 | 21.09 | 6.424 | 69.5%
300 | 24.674 | 8.272 | 66.5%
350 | 28.639 | 8.862 | 69.1%
400 | 33.42 | 10.352 | 69.0%
450 | 37.39 | 12.02 | 67.9%
500 | 39.944 | 13.288 | 66.7%
550 | 45.019 | 15.005 | 66.7%
600 | 48.237 | 15.349 | 68.2%
650 | 53.304 | 17.149 | 67.8%
700 | 56.961 | 18.124 | 68.2%
750 | 61.672 | 19.079 | 69.1%
800 | 66.185 | 20.485 | 69.0%
850 | 69.524 | 21.584 | 69.0%
900 | 74.754 | 22.304 | 70.2%
950 | 79.493 | 25.932 | 67.4%

![Comparison of remote call time with varying numbers of Actors and object ref parameters in remote calls](https://github.com/ray-project/ray/assets/11072802/89a5a0c4-3dfe-4fae-b046-0e1c72790fe1)

**Conclusion:**  
After comparison, in the scenario of remote calls with object ref paramter, the performance is optimized by 3~4 times.

**Summary:**  
The newly added Batch Remote API can improve performance in the case of batch calling Actor task. It can reduce performance costs such as parameter serialization, object store consumption, and Python and C++ execution layer switching, thereby improving the performance of the entire distributed computing system.  
Especially in the following scenario:  
1. large parameters 
2. a large number of Actors


### Failure & Exception Scenario.

**1. Exceptions occurred during parameter validation or preprocessing before batch submission of ActorTasks.**
Since these exceptions occur before the process of submitting ActorTasks, they can be handled by directly throwing specific error exceptions as current situation.

**2. Some actors throw exceptions during the process of batch submitting ActorTasks.**
When traversing and submitting ActorTasks in a loop, if one of the Actors throws an exception during submission, the subsequent ActorTasks will be terminated immediately, and the exception will be throwed to user.

Reason:  
1. Submitting ActorTask is normally done without any exceptions being thrown. If an error does occur, it is likely due to issues with the code and will require modifications.
2. The exception behavior of this plan is the same as the current foreach remote.

## Compatibility, Deprecation, and Migration Plan
N/A

## Test Plan and Acceptance Criteria
1. The basic test cases.

## (Optional) Follow-on Work

### 1. Implement Ray's native collective communication library through this interface.

The collection communication of Ray's CPU computing scenario is currently implemented through Gloo. However, there are two issues with the collection communication implemented through Gloo:
1. Users need to require other dependency(Gloo), which increases the usage cost.
2. According to our testing, Gloo has many problems when it comes to supporting large-scale scenarios with more than 400+ nodes.

Therefore, we want to develop a native collection communication library for Ray with the aim of making it more convenient for users and supporting large-scale scenarios.  
This native library will undoubtedly rely on the current batch remote API to improve performance. After the API is completed, we will consider implementing the native collection communication library for Ray.


### 2. ActorPool adds the capability of batch submitting ActorTask.
ActorPool is a utility class to operate on a fixed pool of actors. 
This feature can be added to this utility class.
```
a1, a2 = Actor.remote(), Actor.remote()
pool = ActorPool([a1, a2])
refs = pool.batch_remote().task.remote()
```