## Summary

### General Motivation

Ray clusters rely on the Global Control Store (GCS) as the centralized metadata store and cluster manager. In High Availability (HA) deployments, the GCS state is backed by an external persistent store (e.g., Redis). When the primary head node fails, standard recovery mechanisms involve provisioning a new head pod from scratch. This introduces significant latency (image pulling, container startup, cluster re-registration) and risks disconnecting active worker nodes due to prolonged GCS downtime.

To minimize downtime, the **Active-Passive Head** architecture introduces a hot standby (passive) head node that runs alongside the primary. This enhancement proposal details the implementation of a **Native Leader Election** mechanism embedded directly within the C++ GCS process. By using [Time-bounded Leases](https://kubernetes.io/docs/concepts/architecture/leases/), the passive head can rapidly detect failures, acquire the lease, and promote itself to active leader in seconds.

Historically, Ray components have assumed a strict single-head node presence. This design explicitly breaks this assumption, laying the ground work for true HA capabilities.

**Time-Bounded Leases**: A distributed lock pattern where a candidate acquires a time-limited lock (a "lease"). The leader must continuously renew the lease before it expires. If renewal fails, the lease drops, allowing other candidates to contend for ownership.


### Should this change be within `ray` or outside?

This lease-based leader election change should be within Ray core (specifically the C++ GCS server) to ensure fate-sharing, minimize external dependencies, and optimize state transition performance.

We also need some KubeRay implementation to support this change.

It is designed to be an opt-in feature, controlled by an environment variable.

The initial design and implementation targets **KubeRay** exclusively for the following reasons:
- **Standardized Production platform**: The majority of business critical high-availability Ray workloads are currently running on KubeRay.
- **Simplicity**: Leverages existing, strongly consistent Kubernetes native lease objects, avoiding the complexity of embedding a heavy C++ consensus framework (like Raft) into Ray core.
- **Automated Recovery**: Leverages Kubernetes' built-in awareness of Pod availability and node health to quickly promote a standby head node in case of failures.

## Stewardship

### Required Reviewers
- @MengjinYan
- @andrewsykim
- @edoakes

### Shepherd of the Proposal (should be a senior committer)
- @edoakes

## Design and Architecture

### Cluster Lifecycle & Scenarios

#### Scenario A: Cluster Initialization
1. **Startup**: KubeRay provisions two head pods simultaneously (Active and Passive).
2. **Election**: Both pods attempt to acquire the Kubernetes Lease (e.g., `ray-gcs-leader-lock`).
3. **Active Node**: The winner acquires the lease, initializes its GCS managers, loads data from Redis, and starts the full GCS gRPC server.
4. **Passive Node**: The loser enters **Observer Mode**:
   - Starts the GCS gRPC server (`rpc_server_`) in **standby mode** (allowing only the health check endpoint to succeed).
   - Defers loading Redis data.
   - Enters a continuous background loop polling the lease status.

#### Scenario B: Active Head Node Failure
1. **Detection**: The active head pod crashes or becomes unresponsive.
2. **Lease Expiration**: The active head pod fails to renew the lease, and the lease expires after the configured TTL.
3. **Promotion**:
   - The passive head successfully acquires the lease.
   - Triggers `DoStartLoading()` to load the latest cluster state from Redis.
   - Promotes itself to the primary leader and accepts write/read traffic.
4. **Reconciliation**: KubeRay provisions a new head pod, which becomes the new passive head.

#### Scenario C: Passive Head Node Failure
1. **Impact**: Zero impact on the active cluster. The active leader continues to hold the lease and serve traffic.
2. **Reconciliation**: KubeRay provisions a new passive head pod.

#### Scenario D: Both Head Nodes Failure
1. **Impact**: The cluster has no active head and experience longer recovery latency
2. **Recovery**: It falls back to standard fault tolerance for the single head pod. KubeRay initializes two new head pods to begin a leader election, mirroring the initial cluster startup. Once elected, the active head utilizes the standard fault tolerance process to restore GCS metadata from the external Redis instance.

#### Scenario E: Kubernetes API Server Disconnection
1. **Transient Network Disconnection**: If the active leader cannot renew the lease due to network flaps, it relies on the remaining `LeaseDuration`. If the timeout is reached without renewal, the leader **must aggressively terminate (`RAY_LOG(FATAL)`)** to prevent split-brain.
2. **Complete Outage**: If the central K8s control plane goes offline, neither pod can modify leases. The standby remains passive. The active leader may optionally continue running to maintain availability, acknowledging that failover is impossible.
3. **Tradeoff**: It is the tradeoff between availability and safety. From the active head perspective, it has no idea about if the current disconnection is an APIServer outage or just its own network partition. Longer lease duration can avoid frequent leader transition.

### GCS Native Leader Election

We propose integrating the lease election protocol directly into the GCS C++ runtime. 

#### 1. Protocol

The architecture relies on an **Active-Passive** model with distinct responsibilities:
- **Active Node (Leader)**: Holds the Kubernetes Lease, initializes all GCS managers, hydrates the cluster state from Redis, and fully serves read/write traffic. It is responsible for continuously renewing the lease.
- **Passive Node (Standby)**: Operates in a restricted passive mode. It runs the GCS gRPC server in a read-only state (to satisfy health checks) but defers loading cluster state and isolates itself from external traffic routing. It continuously polls the lease status.

For Ray on Kubernetes, these two pods will be scheduled on different nodes by leveraging `podAntiAffinity` configuration to prevent a single node failure from taking down both the active and passive heads.

The **Promotion Protocol** is designed to be non-blocking and safe:
1. **Background Polling**: A dedicated thread continuously polls the lease status via the Kubernetes API, completely isolating blocking network I/O from the main ASIO event loop.
2. **Fail-Fast on Leadership Loss**: Continues to poll the lease state post-promotion. If lease ownership lapses or renewal requests cross the expiration deadline, the leader immediately self-terminates (`RAY_LOG(FATAL)`) to protect cluster state consistency and avoid split-brain scenarios.
3. **State Transition**: When leadership is acquired, the background thread posts a promotion event to the main `io_context`.
4. **Data Loading**: The main thread executes `DoStartLoading()`, reading from Redis and populating the local GCS tables.
5. **Service Activation**: Once data is loaded, the GCS managers are initialized, and the node transitions from standby to active status.


#### 2. Lease Client Implementation (`src/ray/gcs`)

- **`LeaderLeaseClientInterface`**: Platform-agnostic interface defining lease operations (`TryAcquire`, `Renew`).
- **`K8sLeaseClient`**: Implementation utilizing `libcurl` to execute REST payloads against the Kubernetes Coordination API (`/apis/coordination.k8s.io/v1/namespaces/{namespace}/leases/{name}`).
- **`LeaderLeaseClientFactory`**: Decouples the concrete REST client from GCS application initialization paths.

**New Environment Variables:**
- `RAY_LEADER_ELECT`: Enables native leader election for the GCS process.
- `RAY_LEADER_ELECT_LEASE_DURATION`: The duration that non-leader candidates will wait before forcing an acquisition of leadership (default: 15s).
- `RAY_LEADER_ELECT_RENEW_DEADLINE`: The acting leader's bounded deadline for executing consecutive renewal sequences before voluntarily stepping down (default: 10s).
- `RAY_LEADER_ELECT_RETRY_PERIOD`: The duration clients wait between sequential resource attempts (default: 2s).
- `RAY_LEADER_ELECT_RESOURCE_NAME`: The name identifier of the target Kubernetes Lease object.
- `RAY_LEADER_ELECT_RESOURCE_NAMESPACE`: The operational namespace mapping the lease scope.

#### 3. Request and Traffic Routing
- **[Existing]Kubernetes Service for Worker Discovery**: 
  - **Stable DNS Endpoint**: KubeRay provisions a static Head Service (e.g., `<cluster-name>-head-svc`) acting as the central, unchangeable discovery address for all worker pods. Workers configure their startup parameters to target this Service DNS name (`--address=<cluster-name>-head-svc:6379`) instead of ephemeral pod IPs.
- **Readiness Probe Control**: 
  - The default KubeRay `readinessProbe` for the head node is usually configured to use an `exec` probe checking the local Dashboard API endpoints:
    `wget -T 2 -q -O- http://localhost:52365/api/local_raylet_healthz | grep success && wget -T 10 -q -O- http://localhost:8265/api/gcs_healthz | grep success`. We should extend the `gcs_healthz` endpoint to check the leadership status. It should return 200 OK if GCS is the active leader AND is healthy.
  - In **Passive Mode**, the readiness probe explicitly fails, removing the pod from Service endpoints.
  - Upon **Promotion**, the readiness probe succeeds, routing traffic to the new leader.
  - This mechanism prevents clients and workers from attempting to communicate with a standby GCS that has deferred loading its state.
- **Seamless Failover Discovery**: During a transition, the unready standby pod is physically excluded from the Service endpoints. Once promoted, the new leader passes readiness checks, instantly updating the Service endpoints. Consequently, reconnecting workers automatically discover the new active GCS instance via the identical DNS address without requiring configuration updates or container restarts.
- **Head Node Internal Traffic**: Processes on the head pod (e.g., Autoscaler, Dashboard) must be configured to connect **directly to the local GCS instance** (e.g., via `127.0.0.1`) rather than the Kubernetes Service address to avoid cross-pod state confusion.

#### 4. Split-Brain Safeguards

The most severe risk in any HA election architecture is the **Split-Brain** issue—a condition where multiple GCS pods concurrently act as the primary leader. Even though Kubernetes handles the infrastructure-level lock perfectly via etcd's strong consistency, application-level split-brain can still occur. In a multi-threaded C++ environment, the root cause is almost always time blindness—when your application loses track of real-world time due to being blocked or suspended.

There are two scenarios that could suffer from this issue:

##### 1. Network Partition
A network partition occurs while the Leadership Renew Thread is trying to contact the K8s API server. The HTTP client hangs indefinitely waiting for a response. Because the thread is stuck, it cannot check its internal clock, misses the `RenewDeadline`, and fails to trigger a process suicide before K8s elects a new leader.

##### 2. Process Frozen
The host machine experiences severe CPU starvation, or the OS forcefully pauses your entire GCS process. The K8s lease expires, and a new leader takes over. When the OS finally unfreezes the process, the GCS main thread instantly executes some ongoing resource update and database writes before the renew thread has the microsecond needed to call `fatal()` and kill itself. Enabling [resource isolation with Cgroup v2](https://docs.ray.io/en/latest/ray-core/resource-isolation-with-cgroupv2.html) can help with this issue.

##### Safeguards & Mitigations

###### Phase 1: Generic Production Golden Standards
We will apply the following mechanisms, which are production golden standards for leader election implementations. 

1. **Time-bounded parameters configuration**: To ensure safe failovers and prevent 99% of the split brain problem, the timing variables(LeaseDuration, RenewDeadline, HTTP Timeout, RetryPeriod) should align with each other. The following shows the formula for best practice:
	- `LeaseDuration > RenewDeadline > RetryPeriod`;
	- `LeaseDuration > HTTP Timeout + RenewDeadline`;
	- `RenewDeadline > RetryPeriod * factor(3 or 5)`;

	Example: LeaseDuration - 15s; RenewDeadline - 10s; HTTP Timeout - 2s; RetryPeriod - 2s;
	
	**Note**: The longer the LeaseDuration, the more resilient the cluster is to temporary network glitches, but the longer it takes for a failed node to be considered offline.

2. **Fencing Token**: The last barrier to prevent data inconsistency. It is also one of the best practices for implementing lease-based leader election in production. At the end of the day, we must make sure the data stored in redis is pure.
	- **Sourcing the Token**: The active GCS uses the `spec.leaseTransitions` field from the Kubernetes `Lease` object as its token. This is a monotonic integer managed by the K8s control plane and incremented automatically upon every single transition of lease ownership.
	- **Token Persistence**: A dedicated key `GCS_LEADER_EPOCH` is maintained in Redis for concurrency comparison, tracking the value of `spec.leaseTransitions`.
	- **Atomic Updates (Fencing on Redis)**: All writes from GCS to Redis are routed through thin atomic Lua scripts. The Lua script validates that the GCS client's token is at least as large as the active `GCS_LEADER_EPOCH`. If the GCS client acts stale (smaller token epoch), the request fails, preventing any potential split-brain zombie writes. If greater, it overrides the stored epoch and commits the write.
	  ```lua
	  -- Example of fencing lua script
	  local current_epoch = tonumber(redis.call('GET', 'GCS_LEADER_EPOCH') or '0')
	  local client_epoch = tonumber(ARGV[1])

	  if client_epoch < current_epoch then
	      return redis.error_reply("FENCED: stale epoch")
	  end

	  redis.call('SET', 'GCS_LEADER_EPOCH', client_epoch)
	  return redis.call(ARGV[2], unpack(KEYS), unpack(ARGV, 3))
	  ```

###### Phase 2: Ray-specific Optimizations (Out of scope for initial implementation)
Besides those generic safeguards, we can also apply the following Ray-specific optimizations to enhance the GCS HA scenario. Techenically, they could provide additional data consistency guarantees. However, it may involve other potential risk and additional implementation work.(e.g. communication between heads/workers could be unreliable) We will leave them to future iterations. 

1. **Cross-head communication**: Establish the connection between heads. The passive cannot be promoted until it notices that the previous head is stepped down.

2. **Worker Awareness**: The worker nodes track the GCS leader’s state(e.g. Pod name, generation ID…). If the local worker receives requests issued by a stale ID or unrecognized head, it drops the RPC and ignores the request. 
##### Metrics and Monitoring
- `gcs_is_leader`: Gauge metrics. Emits a 1 if the current instance is the leader, and a 0 if it is a passive head.
- `gcs_leader_transitions_total`: Counter metrics. Incremented every time a leadership change is detected. For a single GCS process, the metrics will only ever transition from 0 to 1. Next time a failover occurs, a brand-new GCS pod will spin up, start at 0, and then transition to 1 when it becomes the active leader. 
- `gcs_recovery_latency_ms`: Histogram metrics. The time it takes for a passive head to recover the state from the moment that it takes over the leadership.
- `gcs_lease_renew_latency_ms`: Histogram metrics. The latency of GCS leader lease renewal request.
- `gcs_lease_renew_failures_total`: Counter metrics. The total number of failed GCS leader lease renewal requests.

Even though the passive head is in "read-only" mode and failing readiness probes (so it doesn't get traffic), its metrics endpoint should still be active. An external dashboard should aggregate and plot these metrics for both pods, allowing users to see the health of both the active and passive heads.  

##### Detection and Alerting 
The cluster-wide metrics aggregation and alert should be built outside of gcs/ray process. Here are some examples:
  - Alert if `gcs_is_leader` is 1 for both heads(split brain).
  - Alert if `gcs_recovery_latency_ms` is continously too high.
  - Alert if `gcs_leader_transitions_total` is too frequent(leader flapping).
  - Alert if `gcs_lease_renew_failures_total` rate is high (potential lease loss risk).
  - Alert if P99 of `gcs_lease_renew_latency_ms` is close to `RenewDeadline`.

#### 5. KubeRay Changes

To support the Active-Passive Head architecture, the KubeRay operator requires updates to its CRDs and controller reconciliation logic.

##### 1. Customer-Facing API Changes

Currently, the `HeadGroupSpec` in the `RayCluster` CRD assumes a single head node instance. To support Active-Passive High Availability, we propose adding an `EnableActivePassiveHead` boolean field under `GcsFaultToleranceOptions` to control the active/passive mode enablement. If set to `true`, KubeRay will automatically provision a standby head node. To streamline leader election tuning, we also extend `GcsFaultToleranceOptions` to natively support lease parameters.

**Go API Schema Definitions (`apis/ray/v1/raycluster_types.go`):**

```go
type GcsFaultToleranceOptions struct {
	// RedisConfiguration struct defining external storage endpoints
	...

	// EnableActivePassiveHead enables active-passive high availability for the GCS.
	// If enabled, KubeRay will provision a standby head node to ensure quick recovery.
	// +kubebuilder:default:=false
	EnableActivePassiveHead *bool `json:"enableActivePassiveHead,omitempty"`

	// LeaderElectionLeaseDurationSeconds is the duration that non-leader candidates wait before forcing leadership acquisition.
	// +kubebuilder:default:=15
	LeaderElectionLeaseDurationSeconds *int32 `json:"leaderElectionLeaseDurationSeconds,omitempty"`

	// LeaderElectionRenewDeadlineSeconds is the acting leader's bounded deadline for executing consecutive renewal sequences.
	// +kubebuilder:default:=10
	LeaderElectionRenewDeadlineSeconds *int32 `json:"leaderElectionRenewDeadlineSeconds,omitempty"`

	// LeaderElectionRetryPeriodSeconds is the duration clients wait between sequential resource acquisition attempts.
	// +kubebuilder:default:=2
	LeaderElectionRetryPeriodSeconds *int32 `json:"leaderElectionRetryPeriodSeconds,omitempty"`
}
```

**Validation Webhook:**
- If `EnableActivePassiveHead` is set to `true`, the webhook validates that external Redis persistence is configured under `spec.gcsFaultToleranceOptions` (since native GCS recovery relies on Redis storage).
- Enforces timing invariants if customized: `LeaderElectionLeaseDurationSeconds > LeaderElectionRenewDeadlineSeconds > LeaderElectionRetryPeriodSeconds`.
- Validates that lease parameters (`LeaderElectionLeaseDurationSeconds`, `LeaderElectionRenewDeadlineSeconds`, `LeaderElectionRetryPeriodSeconds`) are **only** configured when `EnableActivePassiveHead` is set to `true`.

**Example Customer Manifest (`RayCluster` CR):**

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  name: raycluster-ha-sample
  namespace: default
spec:
  rayVersion: '2.9.0'
  gcsFaultToleranceOptions:
    redisAddress: "redis:6379"
    enableActivePassiveHead: true # Enable Active-Passive HA
    leaderElectionLeaseDurationSeconds: 20
    leaderElectionRenewDeadlineSeconds: 12
    leaderElectionRetryPeriodSeconds: 3
  headGroupSpec:
    rayStartParams:
      ...
    template:
      spec:
        containers:
        - name: ray-head
          image: rayproject/ray:2.9.0
```

##### 2. Controller Configuration & Reconciliation Logic

When a `RayCluster` is deployed with `gcsFaultToleranceOptions.enableActivePassiveHead: true`, the KubeRay controller executes the following advanced reconciliation logic:

- **Head Pod Provisioning & Lifecycle**:
  - Provisions two independent head pods labeled as candidates for leadership.
  - Automatically maps the fields configured under `gcsFaultToleranceOptions` into the required environment variables inside the head containers to trigger native C++ GCS leader election:
    - `RAY_LEADER_ELECT=true`
    - `RAY_LEADER_ELECT_RESOURCE_NAME=<cluster-name>-gcs-leader-lock`
    - `RAY_LEADER_ELECT_RESOURCE_NAMESPACE=<pod-namespace>`
    - `RAY_LEADER_ELECT_LEASE_DURATION=<value>`
    - `RAY_LEADER_ELECT_RENEW_DEADLINE=<value>`
    - `RAY_LEADER_ELECT_RETRY_PERIOD=<value>`
- **Service Routing & Readiness Control**:
  - Modifies the Ray Head Service reconciliation to explicitly set `publishNotReadyAddresses: false` (overriding default behaviors where head services might publish unready addresses).
  - Configures the Kubernetes Head Service to map its endpoints strictly based on pod readiness, external and worker traffic is dynamically routed exclusively to the active leader pod.
- **Internal Component Routing**:
  - Co-located system components running inside the head pod (such as the Autoscaler, Dashboard, and Dashboard Agent) must be configured to communicate with the local loopback GCS interface (`127.0.0.1:6379`) rather than the shared Kubernetes Head Service DNS name. This prevents cross-pod pollution where a background process on the passive head communicates with the active head's GCS.
- **Automatic Pod Anti-Affinity Injection**:
  - To prevent a single-node outage from terminating both head pods simultaneously, the controller inspects the head pod template. If no explicit affinity rules are defined, it automatically injects a `podAntiAffinity` policy targeting the `ray.io/node-type: head` label to schedule active and passive heads on different nodes.
## Compatibility, Deprecation, and Migration Plan

- **Head Node Components Compatibility**: 
  To ensure a highly available Active-Passive architecture, all background processes running on the Ray Head Node must operate safely in a multi-head environment. Specifically, all components must adhere to the following design principles:
  1. **Suppress Unnecessary Work in Passive Mode**: Standby components must disable non-essential processes (e.g., registering nodes with the GCS, accepting job requests, publishing/aggregating cluster events).
  2. **Avoid Interfering with the Active Leader**: Standby components must not perform tasks reserved for the active head node. For instance, only the active leader's autoscaler should update KubeRay CRDs, and only the active GCS server should persist states to Redis.
  3. **Recover Functionality Automatically Post-Promotion**: Standby components must continuously monitor the leadership status (e.g., via polling periodically) and automatically resume full operations immediately upon promotion. While some components like Raylet and Autoscaler already have the channel to fetch status from local GCS periodically, we need to ensure that the leadership status is propagated to all components. The GCS needs to expose an rpc for other components to fetch its leadership status.

  **1. Single-container co-located components**:
  These components are running in the same container as different subprocesses. The gcs termination caused by lease lost will terminate other processes.
  - **GCS Server**
    Safe on Standby. Global tracking managers are instantiated empty; state hydration is deferred until promotion. The rpc server can only answer critical read requests in passive mode including `GetClusterId` for head initialization and the liveness & readiness health check rpc. All mutation requests and unnecessary read requests should not be accepted and the rejection should be handled gracefully on the client side. (With the proper traffic routing, the requests will be sent to the active head only. But it is another layer of protection to avoid any potential split brain issue.)
    - **Leadership Loss**: If the active GCS fails to renew its lease or voluntarily steps down, it immediately self-terminates (`RAY_LOG(FATAL)`) to eliminate split-brain risks, instantly closing all active gRPC client streams.
  - **Raylet**: it is required to be alive to answer the readiness and liveness probes.
      - **Startup**: Upon boot, the local standby Raylet targets the local GCS and sends a RegisterNode gRPC request. On passive head, the PUT request should not reach to Redis. 
      - **Passive State**: The Raylet continuously sends heartbeats to the local GCS to keep its node liveness. 
      - **Promotion**: During promotion, the Raylet should register itself and  physically writes it to Redis.
  - **Dashboard**: the local Dashboard process is started by KubeRay pointing to the local GCS. Because it queries the passive local GCS, the dashboard UI initially displays a single-node cluster (only itself) with no active jobs or actors. Since the standby pod is Not Ready, its dashboard API is completely unreachable by external clients with the current k8s service setup. 
  - **Job API**: with right k8s service configuration, the `ray job submit` request should not be able to reach the passive head. If it does, the server should reject the request with a well-defined error message and code under passive mode.
  - **Serve Controller**: The Ray Serve Controller is a standard Ray actor. Since GCS in standby mode has no active workers and no running jobs, no Serve Controller is ever spawned on the standby head.
  - **Dashboard Agent**: Runs locally and reports events to the co-located passive GCS. Upon promotion, it continues reporting to the now-promoted local GCS. Ready to initialize runtime environments if new jobs are submitted post-failover. Block gcs event publishing in passive mode.

  **2. Sidecar components**
    - **Autoscaler**: In standard KubeRay setups where `enableInTreeAutoscaling: true` is configured, the Autoscaler is typically injected as a dedicated sidecar container running alongside the ray-head container inside the same pod. It is a critical, high-risk background component that must be suppressed on passive head.
      - **Startup**: Upon boot, the local autoscaler targets the local GCS and periodically sends a `GetClusterResourceState` gRPC request to check if the cluster resource state in GCS is ready. The cluster resource state should only be ready when the GCS is in active mode.
      - **Active State**: Autoscaler gets the cluster resource state from GCS and continues with the normal autoscaling logic including updating KubeRay CRDs, reporting the autoscaling state back to GCS.
      - **Passive State**: Autoscaler periodically gets the cluster resource state from GCS, which indicates the cluster resource state is not ready. The autoscaler skips the rest of the logic. The polling interval is configured by `AUTOSCALER_UPDATE_INTERVAL_S` (default: 5s).
      - **GCS Failure**: If the active head GCS fails, the gRPC call `GetClusterResourceState` will raise an error, causing the local autoscaler to skip the rest of the logic and retry at the interval specified by `AUTOSCALER_UPDATE_INTERVAL_S`.
      - **Promotion**: Once the GCS of the passive head is promoted to active, its local autoscaler should detect that the cluster resource state is ready via the `GetClusterResourceState` gRPC call and start the normal autoscaling logic.

## Baseline latency
By default, we adhere to the standard `client-go` leader election settings:
- `LeaseDuration`: 15s
- `RenewDeadline`: 10s
- `RetryPeriod`: 2s

Here is the latency profiled for 30 failover iterations:

```
Metric                    Min (s)    Max (s)    Average (s)
------------------------------------------------------------
Leadership Acquisition    17         20         18.67
Total Recovery (Worker)   18         28         20.80
```

## Test Plan and Acceptance Criteria

- **Unit Testing**: Verify REST client retry behaviors, exponential backoff, and timeout bounds.
- **E2E Testing**: Simulate GCS failover scenarios and validate cluster failover completes successfully (no data loss, no worker node restarts).
- **Chaos Testing**: Execute simulated pod evictions, GCS OOM/CPU starvation, network partitions isolating the leader, and API server crashes. Validate that GCS failover finishes successfully.

## Follow-on Work

1. **Application-level data consistency guarantees**: Provide additional consistency guarantees beyond the standard mechanisms. Check the details above at the split-brain safeguards section.
2. **Pre-heated passive head with memory**: The passive head continuously pulls/syncs state from the primary gcs. It could be done via the existing pub/sub mechanism. This pre-heated memory synchronization should be deferred until the core active-passive head architecture is established. Improperly managed data synchronization could lead to critical issues such as split-brain or time-travel. Another detailed design is required specifically.
