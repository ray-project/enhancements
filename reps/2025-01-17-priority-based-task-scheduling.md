# Priority Based Task Scheduling

## Summary

Introduce task priority concept into Ray scheduling and a prioritizer plugin system.

### General Motivation

Ray scheduler currently treats all tasks the same with no priority difference. However there are user cases where people want to assign priority to tasks and want high priority tasks to be executed first (e.g. https://github.com/ray-project/ray/issues/16782, https://github.com/ray-project/ray/issues/6057).

### Should this change be within `ray` or outside?

Inside `ray` project since this is a Ray Core feature.

## Stewardship

### Required Reviewers

@edoakes, @stephanie-wang, @pcmoritz, @robertnishihara

### Shepherd of the Proposal

@edoakes

## Design

To support different ways to decide task priority (e.g. static, dynamic based on runtime), we propose a prioritizer plugin system. The Prioritizer interface is 

```
class Prioritizer:

   Prioritizer(const RuntimeInfoProvider &runtime_info_provider)

   string name()

   // Calculate the priority for the given task
   int calculate_priority(task)

   // Whether we need to recalculate previously calculated priorities.
   // This will be used by the runtime based dynamic prioritizer
   bool should_recalculate_priority()
```

The prioritizer then decides the priority based on the task itself and the runtime information (e.g. object store utilization) provided by the RuntimeInfoProvider. Task can pass additional infomation to the prioritizer via task lables. After priority is decided by the prioritizer, Ray scheduler will sort all tasks based on priority and schedule higher priority tasks first.

### Examples

#### Staitc prioritizer

A static prioritizer can be implemented as

```
class StaticPrioritizer: Prioritizer:
   string name()
      return "static_prioritizer"

   // User provides the priority directly
   int calculate_priority(task)
      if "plugin.static_prioritizer.priority" in task.labels
          return int(task.labels["plugin.static_prioritizer.priority"])
      else
          0
   
   bool should_recalculate_priority()
      # static priority doesn't change
      return false
```

and task priority can be statically specified via

```
@ray.remote(labels={"plugin.static_prioritizer.priority": "1"})
def task():
  return 1
```

#### Dynamic prioritizer based on object store memory

Assuming we have two types of tasks: producer tasks that generates objects and consumer tasks that consumes those objects. When the object store memory utilization is low, we want to prioritize producer tasks and when object store memory utilization is high, we want to prioritize consumer tasks. This can be achieved by a different Prioritizer implementation:

```
class ObjectStoreMemoryBasedPrioritizer: Prioritizer:
   string name()
      return "object_store_memory_based_prioritizer"

   // Priority based on runtime object store memory utilization
   int calculate_priority(task)
      if "plugin.object_store_memory_based_prioritizer.low_object_mem_priority" not in task.labels:
         return 0
      last_time_object_store_memory_utilization = runtime_info_provider.object_store_memory_utilization
      if runtime_info_provider.object_store_memory_utilization > threshold
        return int(task.labels["plugin.object_store_memory_based_prioritizer.low_object_mem_priority"])
      else
        return int(task.labels["plugin.object_store_memory_based_prioritizer.high_object_mem_priority"])

   bool should_recalculate_priority()
      return (runtime_info_provider.object_store_memory_utilization > threshold) != (last_time_object_store_memory_utilization > threshold)
```

and task can be submitted as 

```
@ray.remote(labels={"plugin.object_store_memory_based_prioritizer.low_object_mem_priority": "1", "plugin.object_store_memory_based_prioritizer.high_object_mem_priority": "10"})
def producer():
  return np.array(10000000)

@ray.remote(labels={"plugin.object_store_memory_based_prioritizer.low_object_mem_priority": "10", "plugin.object_store_memory_based_prioritizer.high_object_mem_priority": "1"})
def consumer(obj):
  return len(obj)
```

### Open question

Currently, Ray scheduler is distributed in the sense that every raylet is making scheduling decision independently for local tasks. However in order to achieve global prioritization, we need a centralized scheduler that sorts all tasks across the cluster based on priority. Moving to centralized scheduler is a big architectural change and requires its own design. Regardless whether we do distributed scheduler or centralized scheduler, the proposed APIs should be the same.

