
## Summary
### General Motivation

Ray does not currently expose public APIs for understanding the state of the system (what is running, resource consumption, errors, etc.). This is a frequent pain point for end-users and developers. This REP proposes to introduce structured APIs as part of the Ray 2.0 API for key states: actors, tasks, scheduling info, objects, resources, logs, etc.

#### Key requirements:
- Need to expose all necessary Ray states for basic observability.
- State APIs should be consistent / stable / well-documented. E.g., APIs names must be consistent to the concepts
- APIs should be available to all clients, such as Dashboard, CLI, and Python APIs with the same implementation.
- Shouldn’t introduce high overhead to the normal workloads.
- Under no circumstances should it be possible to crash a cluster (say, up to our scalability envelope) using the observability APIs.


### Should this change be within `ray` or outside?

Accessing state is a fundamental feature to debug systems. These changes would lie within Ray to improve the basic observability of the project.

## Stewardship
### Required Reviewers

@scv119, @edoakes, @ericl

### Shepherd of the Proposal (should be a senior committer)

@edoakes

## Design and Architecture

### States
“States” must be closely related and consistent to “concepts / APIs” users learn. For example, if we expose tasks as core Ray concepts, users would like to see the state of tasks. Ideally, the name of the APIs should be consistent with the concepts too.

State APIs can be categorized by 3 types.

**Logical states**
- Aligned with logical concepts in which users need to learn to use Ray.
- Useful from beginners to advanced users.
- E.g., Actor, Task, Objects, Resources, Placement Group, Runtime Env

**Physical states**
- States from physical components.
- When users scale up, it will become important to observe physical states. 
- Logs, Cluster Health, Worker processes, Nodes

**Internal states**
- Internal information of Ray.
- They are mostly useful for maintainers to debug bugs, but users can also be benefited. 
- Scheduling states, Events, Internal states of Ray components 


### Status quo
The below table shows if the existing APIs support the states above. Each column explains if the API is available for each client (Python API, CLI, REST API). Each row indicates available states in Ray. 

TODO(sang): Add a picture.

### Proposed APIs
All states should be visible to the users through properly documented / stable / consistently named APIs. This section proposes the state APIs to observe the following resources. 

- Green: Already available. Might need to make APIs public/stable. 
- Red, P0: States that cannot be accessed / hard to access now
- Yellow, P0: API exists, but we need refinement or renaming.
- Blue, P1: Less critical states. Could be missing or not refined.

TODO(sang): Add a picture

### Terminologies
- Consumer: Consumers of state APIs. In Ray, (e.g., dashboard, CLI, and Python APIs).
- API server: Currently a Ray dashboard. Centralized server to aggregate and provide state information of the cluster.
- Ray agent: Currently a dashboard agent. A Python process running per node.
- Resources: Objects created by Ray. E.g., tasks, actors, placement groups, namespace. 

### Observability API architecture
Currently, there are two big architectural problems.
1. Each consumer has its own implementation for each API. E.g., ray.state.actors() and the API /logical/actors have their own implementation. each consumer in Ray uses different components/code to implement each layer, which in turn leads to having duplicated code and non-unified abstraction, which causes the unmaintainability of state APIs.
2. No architectural pattern to guarantee stability. E.g., ray memory will just crash if the output is big.

We propose to use the Ray API server (which is known as dashboard server) as a stateless, read-only, centralized component to provide state information across Ray. An API server will be responsible for providing interface (either gRPC or REST, depending on what API server is standardized), caching, querying data from sources (raylet, gcs, agent, workers), and post-processing the aggregated state information. Having a centralized component with standardized interface will bring several benefits such as (1) all consumers can avoid having diverged implementation, (2) APIs will become language agonistic, (3) and good maintainability and low complexity with the expensse of performance. Since state APIs don't have strict performance requirements, the tradeoff wouldn't be a big problem.

### Stability and Scalability
Aggregation of large data to a centralized component with reasonable SLA is a hard problem to solve, especially for systems like Ray that has decentralized data sources (tasks, objects). Building systems that can scale well in such scenario would be difficult and might require us to build/introduce new systems. For the simplicity, we proposes to **degrade the performance gracefully** when there are significantly high volume of data (e.g., shuffling millions of objects, and users want to access the list of all objects).

To support stability in the large-scale cluster, we will ensure to bound the output size of API. More concretely, there will be 4 rules. 
- O(1) overhead per node per call (e.g., lim 1000 records)
- O(n) overhead on API server per call (n: number of nodes).
- O(1) final result size (e.g., lim 10000 records)
- API server limits the number of concurrent requests.

Note that it means users can lose information when there is a high volume of data. However, debugging issues with extremely high volume of data will hinder user experience or sometimes not even possible. It means we should optimize for **reducing the output size** rather than supporting users to access all high volume of data.

### APIs
We propose to introduce 3 different types of APIs.
- Summary API, which will provide a comprehensive view of states. For example, tasks can be grouped by a task name, and the API can provide the number of running or pending tasks.
- List API, which will list all existing state of resources in the cluster. Summary is sometimes not sufficient, and users should be able to access more fine-grained information. For example, ray list actors will return all actor information. The API will allow users to filter data.
- Get API, which will return a detailed view of a single resource state.

All the APIs will be available through CLI, Python API, and REST endpoints.

```
# CLI
ray summary actors              # Returns the summary information.
ray list actors --state=CREATED # Returns all entries of actors.
ray get actor [id]              # Return the detailed information of a single actor.

# Python
ray.summary_actors()
ray.list_actors(state="CREATED")
ray.get_actor(id)

# REST
/api/v1/actors/summary
/api/v1/actors?state="CREATED"
/api/v1/actors/{id}
```

The detailed API specification will go through Ray API review.

### Error handling
All consumers should have the default timeout. It is the same for all internal aggregation RPCs.

- If the API server is down (or overloaded).
    - The request will simply timeout and users will be notified.
    - The availability of the API server is not in the scope of this REP.

- If the source is down (or overloaded)
    - Return the RPC with data loss and notify users about it.
    - The stability of data sources is not in the scope of this REP.

- The returned payload is too big
    - We will bound the output size of all internal RPCs to make sure this is not happening in the first iteration. In the future, we can solve it by chunking the result or using streaming.

## Compatibility, Deprecation, and Migration Plan
TODO(sang)

## Test Plan and Acceptance Criteria
TODO(sang)

## (Optional) Follow-on Work
TODO(sang)
