
## Summary
### General Motivation

Ray does not currently expose public APIs for understanding the state of the system (what is running, resource consumption, errors, etc.). This is a frequent pain point for end-users and developers. This REP proposes to introduce structured APIs as part of the Ray 2.0 API for key states: actors, tasks, scheduling info, objects, resources, logs, etc.

#### Key requirements:
- Need to expose all necessary Ray states for basic observability.
- State APIs should be consistent / stable / well-documented. E.g., APIs names must be consistent with the concepts
- APIs should be available to all clients, such as Dashboard, CLI, and Python APIs with the same implementation.
- Shouldn’t introduce high overhead to the normal workloads.
- Under no circumstances should it be possible to crash a cluster (say, up to our scalability envelope) using the observability APIs.


### Should this change be within `ray` or outside?

Yes, the states we want to observe belong to Ray's internal components.

## Stewardship
### Required Reviewers

@scv119, @edoakes, @ericl

### Shepherd of the Proposal (should be a senior committer)

@edoakes

## Design and Architecture

### States
“States” must be closely related and consistent with “concepts / APIs” users learn. For example, if we expose tasks as core Ray concepts, users would like to see the state of tasks. Ideally, the name of the APIs should be consistent with the concepts too.

State APIs can be categorized into 3 types.

**Logical states**
- Aligned with logical concepts in which users need to learn to use Ray.
It- Useful for beginners to advanced users.
- E.g., actors, tasks, objects, resources, placement groups, runtime environments


**Physical states**
- States from physical components.
- When users scale up, it will become important to observe physical states. 
- E.g., logs, cluster health, worker processes, nodes

**Internal states**
- Internal information about Ray.
- They are mostly useful for maintainers to debug bugs, but users can also be benefited. 
- E.g., scheduling states, events, internal states of Ray components 


### Status quo
The below table shows if the existing APIs support the states above. Each column explains if the API is available for each client (Python API, CLI, REST API). Each row indicates available states in Ray. 

<img width="826" alt="Screen Shot 2022-04-26 at 11 14 28 AM" src="https://user-images.githubusercontent.com/18510752/165365387-32a11ea4-bd0b-4df5-aeaa-f74e89a5332d.png">
<img width="854" alt="Screen Shot 2022-04-26 at 11 14 13 AM" src="https://user-images.githubusercontent.com/18510752/165365343-0f912117-feda-4050-838d-59bb30c0dd03.png">


### Proposed APIs
All states should be visible to the users through properly documented / stable / consistently named APIs. This section proposes the state APIs to observe the following resources. 

- Green: Already available. Might need to make APIs public/stable. 
- Red, P0: States that cannot be accessed / hard to access now
- Yellow, P0: API exists, but we need refinement or renaming.
- Blue, P1: Less critical states. Could be missing or not refined.

<img width="806" alt="Screen Shot 2022-04-26 at 11 15 05 AM" src="https://user-images.githubusercontent.com/18510752/165365485-25727a08-2a08-4691-a554-fbe5b7544cde.png">
<img width="807" alt="Screen Shot 2022-04-26 at 11 15 17 AM" src="https://user-images.githubusercontent.com/18510752/165365506-1e2c6bc7-c365-4405-a4ca-2c7257051d69.png">

### Terminologies
- Consumer: Consumers of state APIs. In Ray, (e.g., dashboard, CLI, and Python APIs).
- API server: Currently a Ray dashboard. Centralized server to aggregate and provide state information of the cluster.
- Ray agent: Currently a dashboard agent. A Python process running per node.
- Resources: Objects created by Ray. E.g., tasks, actors, placement groups, namespace. 

### Observability API architecture
Currently, there are two big architectural problems.
1. Each consumer has its implementation for each API. E.g., ray.state.actors() and the API /logical/actors have their own implementation. each consumer in Ray uses different components/code to implement each layer, which in turn leads to having duplicated code and non-unified abstraction, which causes the unmaintainability of state APIs.
2. No architectural pattern to guarantee stability. E.g., ray memory will just crash if the output is big.

We propose to use the Ray API server (which is known as the dashboard server) as a stateless, read-only, centralized component to provide state information across Ray. An API server will be responsible for providing an interface (either gRPC or REST, depending on what API server is standardized), caching, and querying data from sources (raylet, GCS, agent, workers), and post-processing the aggregated state information. Having a centralized component with a standardized interface will bring several benefits such as (1) all consumers can avoid having diverged implementation, (2) APIs will become language agonistic, (3) and good maintainability and low complexity with the expenses of performance. Since state APIs don't have strict performance requirements, the tradeoff wouldn't be a big problem.

<img width="359" alt="Screen Shot 2022-04-26 at 11 15 44 AM" src="https://user-images.githubusercontent.com/18510752/165365580-f953181a-67b8-4797-883d-d9b726d2780e.png">

Alternatively, we can allow consumers to directly query the states from each component (which resembles how the private GCS-based state APIs work). Although it is a viable option and could be slightly more scalable (because there's no centralized component), it has several drawbacks. (1) some logic is hard to generalize (e.g., service discovery) from all consumers. (2) APIs will be difficult to be used by some consumers (e.g., dashboard) (3) we will need to develop APIs for each language. 

### Stability and Scalability
Aggregation of large data to a centralized component with reasonable SLA is a hard problem to solve, especially for systems like Ray that have decentralized data sources (tasks, objects). Building systems that can scale well in such a scenario would be difficult and might require us to build/introduce new systems. For simplicity, we propose to **degrade the performance gracefully** when there is significantly high volume of data (e.g., shuffling millions of objects, and users want to access the list of all objects).

To support stability in the large-scale cluster, we will ensure to bound the output size of API. More concretely, there will be 4 rules. 
- O(1) overhead per node per call (e.g., lim 1000 records)
- O(n) overhead on API server per call (n: number of nodes).
- O(1) final result size (e.g., lim 10000 records)
- The API server limits the number of concurrent requests.

Note that it means users can lose information when there is a high volume of data. However, debugging issues with the extremely high volume of data will hinder user experience or sometimes not even possible. It means we should optimize for **reducing the output size** rather than supporting users to access all high volumes of data.

### APIs
Accessing states could be categorized by 2 types of APIs.
- Unique-id-based APIs. For example, actors, tasks, placement groups, nodes, etc. All states could be accessed by similar APIs.
- Other types of APIs that don't have unique ids. For example, logs, resources, or health-checking. These states need to be exposed in their unique way.

Note that the API example below is not finalized, and detailed API specifications will go through the Ray API review.

#### Unique-id based APIs
We propose to introduce 3 different types of APIs.
- Summary API, which will provide a comprehensive view of states. For example, tasks can be grouped by a task name, and the API can provide the number of running or pending tasks. Summary API will help users quickly figure out anomalies and narrow down the root causes of various issues.
- List API, which will list all existing states of resources in the cluster. A summary API is not always sufficient, and users should be able to access more fine-grained information. For example, users would like to access the "state of every single actor" to exactly figure out problematic actors in the cluster.
- Get API, which will return a detailed view of a single resource state. Get API could return more information than list APIs.

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

#### Non unique-id based APIs
We'd like to propose 3 additional APIs that cannot be categorized as unique-based APIs.
- `ray logs`. This API will allow users to access and tail any log files in the cluster through handy CLI commands.
- `ray resources`. This API will allow users to access resource usage of the cluster as well as tasks/actors that occupy those resources.
- `ray health-check. This API will help users to know the health of the cluster. This API already exists as a private API, and we'd like to propose it to be a public API.

The example API and output will look like this. Details will be confirmed as a follow-up.

```
# ray logs
ray logs actor [actor_id] # Print actor logs of the corresponding id.
ray logs raylet [node_id] # Print the raylet logs of the corresponding id.
ray logs raw [filename] # Print the file.
ray logs GCS --follow # Tail the GCS log.

# ray resources
ray resources # Returns the resource usage [used]/[total]. Will look like `ray status`.
ray resources --per-node # Returns the resource usage per node.
ray resources --detail # Returns the list of tasks/actors that use resources
> CPU 4/16, GPU 1/4
>     f, {CPU: 1} * 3
>     Actor, {CPU: 1, GPU:1} * 1

# ray cluster-health # Returns exit code 0 if the cluster is alive.
```

### Error handling
All consumers should have the default timeout. It is the same for all internal aggregation RPCs.

- If the API server is down (or overloaded).
    - The request will simply timeout and users will be notified.
    - The availability of the API server is not in the scope of this REP.

- If the source is down (or overloaded)
    - Return the RPC with data loss and notify users about it.
    - The stability of data sources is not within the scope of this REP.

- The returned payload is too big
    - We will bound the output size of all internal RPCs to make sure this is not happening in the first iteration. In the future, we can solve it by chunking the result or using streaming.

## Compatibility, Deprecation, and Migration Plan
### Dashboard
In the medium term, we'd like to make the existing dashboard use state APIs. The dashboard server has had various stability issues due to its implementation (e.g., never GC some expensive information, or querying data from the cluster inefficiently every 1 second). It also has several permanent states in memory (e.g., a list of actor task specs) which makes it difficult to be fault-tolerant. Once all APIs are implemented, we'd like to propose making the dashboard "use state APIs" on-demand with pagination, so that it will become stateless and efficient.

### API server
We'd like to rename the dashboard server as an API server. It will involve changing various file names and documentation changes. This has been the change that the community has been pushing for, and indeed we've added various dashboard-unrelated features to this component (e.g., job submission or usage states collection).

## Test Plan and Acceptance Criteria
All APIs will be fully unit tested. All specifications in this documentation will be thoroughly tested at the unit-test level. The end-to-end flow will be tested within CI tests. Before the beta release, we will add large-scale testing to precisely understand scalability limitations and performance degradation in large clusters. 

## (Optional) Follow-on Work
- Pagination when accessing states.
- Dashboard migration.
