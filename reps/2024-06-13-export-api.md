## Summary
### General Motivation

In the current design of Ray, the way to export various states in the Ray cluster are inconsistent.
For example, the state of the Actor is broadcast through GCS pubsub, and to obtain the state change of the node,
it is necessary to query rpc service (NodeInfoGcsService). For the jobs submitted through job submission,
there is no way to expose the state. The high-level libraries on top of Ray also don't have a unified export way.
E.g. RayServe and RayData collect the states through their own StateActor and report to the Dashboard respectively.

It is very difficult to obtain these ray basic states outside the Ray cluster. This issue is reflected in two aspects:
1. Scale of data exceeds current dashboard API limits.
2. After the Ray cluster terminates, all status data is lost.
If a unified export API can be defined,
we can achieve the observable ability independent of the Ray cluster. The most typical scenario is to build the Ray history server.

#### Key requirements:
- Need to expose all necessary ray states for basic observability, tasks/actor/jobs/nodes.
- The states export should not increase the load of the ray cluster.
- Export states streamingly rather than fetching it directly from ray cluster. To obtain the current status of the ray cluster at one time,
  you need to use [the state observability api](https://github.com/ray-project/enhancements/blob/main/reps/2022-04-21-state-observability-apis.md "the state observability api") . 
- Certain states can be exported respectively.
- Friendly to all types of users (especially cloud vendors), easy to deploy and use, without modifying Ray.
- It should be possible to disable the export API for security and performance.
- The data should be exported from as close as possible to the source where it is generated. The export API should have minimal overhead and we should not need to unnecessarily move data to other nodes before it is exported.

### Should this change be within ray or outside?

Yes, the states we want to export belong to Ray's internal components.

## Stewardship
### Required Reviewers
@nikitavemuri

### Shepherd of the Proposal (should be a senior committer)

## Design and Architecture
### Event
`Event` represents the state change in the Ray cluster, which indicates that
the state of a certain resource in the Ray Cluster has changed, such as the starting
and stopping of a job, the constuction and destruction of an Actor, etc. Event is structured data,
and each resource corresponds to a specified event type. All types of events follow a generally consistent format,
but the event fields of different types can be customized. E.g.:

```json
{"event_type": "JOB", "job": {"submission_id": "raysubmit_WBMEB9nTaMKmrKrN", "job_info": {"status": "RUNNING", "entrypoint": "python3 my_script.py", "message": null, "error_type": null, "start_time": 1692004435106, "end_time": null, "metadata": null, "entrypoint_num_cpus": null, "entrypoint_num_gpus": null, "entrypoint_resources": null, "driver_agent_http_address": null, "driver_node_id": null}}}

{"event_type": "ACTOR", "actor": {"actor_id": "efc87749b2e337d7872dcf3802000000", "actor_info": {"actorId": "efc87749b2e337d7872dcf3802000000", "jobId": "02000000", "address": {"rayletId": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffff", "workerId": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffff", "ipAddress": "", "port": 0}, "name": "_ray_internal_job_actor_raysubmit_WBMEB9nTaMKmrKrN", "className": "JobSupervisor", "state": "PENDING_CREATION", "numRestarts": "0", "timestamp": 0.0, "pid": 0, "startTime": 0, "endTime": 0, "actorClass": "JobSupervisor", "exitDetail": "-", "requiredResources": {}}}}

```

By aggregating all the Events in time dimension, we can obtain the historical state of the ray cluster,
and the consumers of the events can aggregate the final state from a series of events. The export api
does not guarantee the ultimate consistency of event state, which is feasible and tolerated in production practice.
The loss of some intermediate events will not cause the final state to be incorrect. For example,
if the event of an actor's creation is lost, but the consumer can read the event of actor exiting,
then the consumer knows that the actor finally ran successfully.


The various components of Ray, including GCS, Dashboard, raylet, and worker, can all serve as event generators.
However, these sources don't need to concern themselves with the details of how events are stored, transmitted, or published.

### Events to Export and How
#### Events to Export
- Core
Tasks/Actors/Objects/Jobs/Nodes/Placement groups, including its meta data and current status when event emits.

|  Event            | Event Source          | When to export                                                                    | Format example(maybe a file link)|
|  ----             | ----                  | ----                                                                              | ----                             |
| Tasks             | CoreWorker and raylet | When coreworker adds task event to<br>ray::core::worker::TaskEventBuffer          |          |
| Actors            | GCS                   | GcsPublisher::PublishActor<br>publish actor status through<br>GCS_ACTOR_CHANNEL   |          |
| Objects           | CoreWorker and raylet | None                                                                              |          |
| Jobs              | JobManager            | When JobInfoStorageClient.put_status called                                       |          |
| Nodes             | GCS                   | GcsNodeManager::HandleXXXNode                                                     |          |
| Placement groups  | GCS                   | GcsPlacementGroupManager::HandleXXXPlacementGroup                                 |          |


- RayServe
State change events for replicas, deployments, applications.

|  Event            | Event Source          | When to export | Format example(maybe a file link)|
|  ----             | ----                  | ----           | ----                               |
| Serve App         | ServeController Actor | None           | None         |

- RayData
All datasets, the dag, and execution progress.

|  Event            | Event Source          | When to export | Format example(maybe a file link)|
|  ----             | ----                  | ----           | ----                               |
| Datasets         | ray.data.internal.stats._StatsActor | _StatsActor's update function called.           | None         |


#### How to export
- Generate event
We propose to use the filesystem based export interface. The event source can just write the
event into the file of this node and then return. The export of the event can be the responsibility of the agent on the node. We know that there will be multiple worker processes running multiple actors or tasks on each Ray node. Events generated by multiple worker processes are written to multiple event files, not just one, in order to avoid competition among multiple worker processes or possible file writing errors.


- Export event
We propose to use pubsub for event export, because pubsub is easier to decouple systems,
and it is also pluggable. Various components that support pubsub can be used,
such as GCS in ray cluster, and external systems such as Kafka, etc.
The agent monitors the event file generated on this node and publishes events to the backend.

<br>

<img src="./2024-06-13-export-api-figures/export-api-1.png" alt="alt" title="title" width="600">

<br>


Advantages:
1. For the event sources (gcs/worker/raylet), the export interface is very lightweight and stable,
   and will not encounter abnormal errors such as network timeouts.
2. It decouples the process of the event source generating events and the actual export (publishing to gcs or kafka).
3. It is easy to control and configure. By configuring the agent to control whether to export,
   control the event types that need to be exported, and control the export speed.
4. Using the pubsub mechanism, the event export is streamingly, the publisher and subscriber
   can handle events at their own speed or ignore the events.
5. The mainstream pubsub system (kafka) supports repeated subscription and consumption, which is friendly to subscribers.


#### Usage
Based on the above very flexible solution, users with different needs can choose the most suitable deployment method.
1. Basic use cast: for ordinary users, the scale of the ray cluster and jobs is restricted. It is acceptable
   to utilize the default GCS pubsub. Moreover, for cloud vendors, it is very friendly and important to use open-source
   components easily, and Kafka is a very good choice.
2. Advanced use case: For users with the capability and demand for customized development, such as companies of a
   certain scale that often have a relatively large Ray cluster scale and quantity and possess their own infrastructure.
   When deploying Ray, it is necessary to adapt to their internal publish/subscribe system. By employing the exported
   solution we proposed, only a small amount of modification to the agent is needed. For example, they can set up log
   ingestion (eg: through Vector) on each worker and then exports for their own use cases.


### Status quo
At present, there is already an events mechanism implementation in Ray: components like gcs/raylet/workers
print structured logs (JSON) to the tmp_dir/log/events/ directory, and the event agent on each node
monitors these files and reports the file content to the event head. 
This is also the source of the Events on the Overview page of the dashboard.

<br>

<img src="./2024-06-13-export-api-figures/export-api-2.png" alt="alt" title="title" width="600">

<br>

We propose to reuse the existing Ray event mechanism and build on it.