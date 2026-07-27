# Summary

### General Motivation

Ray is emerging as the de facto orchestrator for Reinforcement Learning (RL) frameworks and AI agent workflows (e.g., veRL, SLIME, NeMo-RL, DeepSeek-R1 style training, coding benchmarks, and tool-use evaluation). A key requirement in these workloads is the safe execution of untrusted, model-generated code and tool calls during training and evaluation loops.

Currently, running untrusted code directly on Ray worker nodes poses severe security and operational risks, including arbitrary host access, data leakage, and cluster instability. To avoid these risks, frameworks often delegate execution to external sandbox services or third-party APIs. However, this externalized approach fragments the Ray ecosystem with ad-hoc abstractions and increases orchestration complexity.

This proposal addresses these challenges by introducing an (experimental) Ray Sandbox library to programmatically manage isolated sandbox environments. The `ray.experimental.sandbox` library will provide a common denominator abstraction with support for widely used backends and unfied resource scheduling with Ray. The initial implementation will support two default sandboxing backends: a Kubernetes-based backend (using K8s Pods) for cluster-managed container isolation, and a gVisor-based backend (`runsc`) operating directly on Ray worker nodes for fast sandbox startup and dense bin packing of isolated execution environments.

### Should this change be within `ray` or outside?

Within `ray` as an experimental library (`ray.experimental.sandbox`).

## Stewardship

### Required Reviewers

@MengjinYan
@richardliaw
@kouroshHakha

### Shepherd of the Proposal (should be a senior committer)

@edoakes @pcmoritz

## Design and Architecture

### Overview

`ray.experimental.sandbox` introduces an abstraction layer between Ray workloads (tasks, actors, or driver scripts) and underlying container or micro-VM runtimes for sandbox environments.

```
+-------------------------------------------------------------------+
|               Ray Application / RL Framework                       |
|           (e.g., veRL, SLIME, Rollout Workers, Agents)            |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                      ray.experimental.sandbox                     |
|  +---------------------+  +-----------------+  +-----------------+|
|  |   Sandbox           |  |  SandboxPool    |  |  ExecResult     ||
|  +---------------------+  +-----------------+  +-----------------+|
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                          SandboxEnv Interface                     |
+-------------------------------------------------------------------+
                 |                                   |
                 v                                   v
+---------------------------------+ +-------------------------------+
|      KubernetesSandboxEnv       | |       GVisorSandboxEnv        |
| (Manages K8s Pods & Security)   | |  (Manages host `runsc` spec)  |
+---------------------------------+ +-------------------------------+
                 |                                   |
                 v                                   v
+---------------------------------+ +-------------------------------+
|        Kubernetes Cluster       | |        Ray Worker Node        |
|  +---------------------------+  | |  +-------------------------+  |
|  | Sandbox Pod               |  | |  | gVisor Sandbox (runsc)  |  |
|  +---------------------------+  | |  +-------------------------+  |
+---------------------------------+ +-------------------------------+
```

### Python API Design

The `ray.experimental.sandbox` library will provide an intuitive, high-level abstractions for creating and managing isolated sandbox environments across common execution backends. The initial implementation will support both Kubernetes Pods and gVisor (`runsc`) sandboxes. Kubernetes Pods offer cluster-managed execution and remote isolation, while gVisor provides node-local container sandboxing optimized for sub-100ms startup times and high-density bin packing of concurrent execution tasks.

#### Example: Basic Sandbox Creation and Command Execution

```python
import ray
from ray import sandbox

ray.init()

# Create a gVisor sandbox environment on the local worker node
sb = sandbox.create(
    runtime="gvisor",
    cpu=1.0,
    memory="512Mi",
    timeout=300,  # Auto-terminate after 5 minutes
)

# Execute code or bash commands inside the sandbox
result = sb.execute("python -c 'import sys; print(sys.version)'")

print(f"Exit Code: {result.exit_code}")
print(f"Stdout: {result.stdout}")
print(f"Stderr: {result.stderr}")
print(f"Duration: {result.duration_ms} ms")

# Clean up resources
sb.terminate()
```

#### Example: Context Manager Usage

```python
from ray import sandbox

with sandbox.create(runtime="gvisor") as sb:
    # Upload local files or scripts into the sandbox Pod
    sb.upload_file(local_path="model_eval.py", remote_path="/tmp/model_eval.py")

    # Run execution
    res = sb.execute("python /tmp/model_eval.py --input /tmp/data.json")

    # Download output artifacts
    sb.download_file(remote_path="/tmp/output.json", local_path="results.json")
```

#### Example: SandboxPool for High-Concurrency RL Training

In RL training workflows (e.g., reward modeling or code execution benchmarks), thousands of code snippets need to be evaluated in parallel across training steps. `SandboxPool` manages a pool of pre-warmed or reusable sandboxes.

```python
import ray
from ray.sandbox import SandboxPool

# Create a pool of 16 Kubernetes sandbox Pods
pool = SandboxPool(
    size=16,
    runtime="kubernetes",
    image="python:3.10-slim",
    namespace="ray-sandboxes",
    reuse_sandboxes=False, # Re-create sandbox per task for strict isolation
)

@ray.remote
def evaluate_generated_code(pool: SandboxPool, code_snippet: str):
    # Acquire a sandbox from the pool
    with pool.acquire() as sb:
        sb.upload_content(content=code_snippet, remote_path="/tmp/test.py")
        res = sb.execute("python3 /tmp/test.py", timeout=10)
        return res.exit_code == 0 and "SUCCESS" in res.stdout

futures = [evaluate_generated_code.remote(pool, snippet) for snippet in code_snippets]
results = ray.get(futures)
```

---

### SandboxEnv Interface

All sandbox implementations will implement a generic `SandboxEnv` interface:

```python
class SandboxEnv(ABC):
    @abstractmethod
    def create(self, config: SandboxConfig) -> str:
        """Provision the sandbox instance and return unique instance ID."""
        pass

    @abstractmethod
    def execute(self, instance_id: str, command: str, timeout: int = None, env: dict = None) -> ExecutionResult:
        """Execute a command inside the specified sandbox."""
        pass

    @abstractmethod
    def upload_file(self, instance_id: str, local_path: str, remote_path: str) -> None:
        """Copy local file into the sandbox."""
        pass

    @abstractmethod
    def download_file(self, instance_id: str, remote_path: str, local_path: str) -> None:
        """Copy file out of the sandbox to local filesystem."""
        pass

    @abstractmethod
    def terminate(self, instance_id: str) -> None:
        """Destroy the sandbox instance and release underlying resources."""
        pass
```

---

### gVisor Env

The `gvisor` sandbox environment implementation (`GVisorSandboxEnv`) uses gVisor's OCI runtime (`runsc`) to execute untrusted code in lightweight, kernel-isolated sandboxes directly on Ray worker nodes.

#### Motivation: Fast Startup & Dense Bin Packing

In RL training and LLM agent evaluation loops, workers execute thousands of short-lived code snippets per second. Traditional Kubernetes Pod creation adds multi-second latency (scheduling, API server overhead, networking setup), creating a bottleneck for high-throughput rollout pipelines.

gVisor addresses these challenges through two key advantages:
1. **Fast Startup**: By intercepting application system calls in user space via the gVisor application kernel, sandbox containers start in tens of milliseconds (sub-100ms latency), bypassing container daemon and Kubernetes control plane overhead.
2. **Dense Bin Packing**: gVisor sandboxes have a minimal memory footprint and near-zero idle CPU overhead per instance. Ray worker nodes can densely pack hundreds or thousands of concurrent gVisor sandboxes alongside Ray actors and tasks without requiring separate VM instances or cluster nodes.

#### 1. Sandbox Provisioning and OCI Spec Generation

When `sandbox.create(runtime="gvisor", ...)` is called on a Ray worker node, `GVisorSandboxEnv` performs the following steps:
- **OCI Bundle Directory**: Creates a unique OCI bundle directory under `/tmp/ray/sandboxes/<instance_id>/` containing a `rootfs` filesystem (extracted from container image or rootfs cache) and an OCI specification `config.json`.
- **OCI Config Generation**: Generates `config.json` (or calls `runsc spec` with standard overrides):
  - **Process Config**: Sets entrypoint process, working directory, UID/GID, and environment variables.
  - **Resource Limits**: Sets cgroup limits for CPU quota (`resources.cpu`) and memory limits (`resources.memory`).
  - **Security Hardening**: Enforces rootless execution (`runAsNonRoot`), read-only rootfs (`readonly: true`), drops capabilities (`capabilities.bounding: []`), and enables seccomp syscall filtering.
- **Sandbox Creation Calls**:
  - `runsc --root=<root_dir> create --bundle <bundle_dir> <instance_id>`: Initializes the gVisor sandbox container and Sentry kernel process without executing the entrypoint yet.
  - `runsc --root=<root_dir> start <instance_id>`: Triggers execution of the sandbox container process inside gVisor.

#### 2. Command Execution Mechanism (`runsc exec`)

To execute commands inside an active gVisor sandbox instance:
- **`execute(instance_id, command, timeout, env)`**:
  - Calls `runsc --root=<root_dir> exec --cwd <cwd> <instance_id> bash -c "<command>"`
  - Connects standard IO streams to capture stdout and stderr directly from the sandbox process.
  - Enforces process timeouts using SIGKILL via `runsc kill` if command execution exceeds the timeout limit.
  - Returns `ExecutionResult` containing `exit_code`, `stdout`, `stderr`, and precise execution `duration_ms`.

#### 3. File Transfer Mechanism

Because gVisor operates on the local worker node, file operations leverage direct filesystem access into the OCI bundle `rootfs`:
- **`upload_file(instance_id, local_path, remote_path)`**: Directly copies the local file into `<bundle_dir>/rootfs/<remote_path>` (or streams via `runsc exec` if strict rootfs isolation is active), achieving microsecond-level copy speeds without network overhead.
- **`download_file(instance_id, remote_path, local_path)`**: Reads directly from `<bundle_dir>/rootfs/<remote_path>` and writes to `local_path` on the host worker filesystem.

#### 4. Lifecycle Management & Garbage Collection

- **Termination**: `sb.terminate()` or context manager exit executes:
  1. `runsc --root=<root_dir> kill <instance_id> SIGKILL`
  2. `runsc --root=<root_dir> delete <instance_id>`
  3. Removes the temporary OCI bundle directory `<bundle_dir>`.
- **Worker Process Death Handler**: Registers exit hooks (`atexit`) and Raylet worker heartbeat checkers to execute `runsc delete -f <instance_id>` and delete stale bundle directories if the parent worker process crashes abruptly.

#### 5. Release / Dependency Updates

Supporting gVisor will require bundling `runsc` into Ray container images. The `runsc` binary is ~30MB.

#### 6. Container Image Support

The initial version of the gVisor runtime will not support specifying container images. As a result, gVisor sandboxes only have access to tools
that are present on the Raylet's root filesystem. However, container images will be supported in the future once the gVisor backend meets baseline
requirements for performance and reliability.

To support container images in a future release, Ray will need to implement the following workflow:
* Pull image from remote registry (e.g. Dockerhub) to the Raylet process's local image cache.
* Untar container image filesystem into a local directory that can be used by gVisor sandboxes
* Support pre-loaded container images in Ray container images for faster sandbox startup

---

### Kubernetes Env

The `kubernetes` sandbox environment implementation (`KubernetesSandboxEnv`) will support creating dedicated Kubernetes Pods to serve as isolated sandbox environments.
This will be achieved through the official Kubernetes Python client or in-cluster service account credentials. For the time being, it will be the user's responsibility
to grant the Raylet's service account the necessary RBAC permissions to create and manage Pods in the target namespace.

#### 1. Pod Provisioning and Configuration

When `sandbox.create(env="kubernetes", ...)` is called, `KubernetesSandboxEnv` will:
- Interact with the Kubernetes API using the official `kubernetes` Python client using the in-cluster service account credentials or local kubeconfig.
- Generates a lightweight Pod specification:
  - **Metadata**: Labels identifying the parent Ray Cluster ID, Ray Job ID, and owner process details.
  - **Resource Allocation**: Sets requested CPU, memory, and ephemeral storage limits based on user parameters.
  - **Security Context (Hardening)**:
    - `runAsNonRoot: true` with non-privileged UID/GID.
    - `allowPrivilegeEscalation: false`.
    - `capabilities: drop: ["ALL"]`.
    - `readOnlyRootFilesystem: true` (with `/tmp` mounted as an `emptyDir` volume).
    - `runtimeClassName`: Supports gVisor (`gvisor`) or Kata Containers (`kata`) when configured on the Kubernetes cluster.
- Apply any user defined modifications to the pod spec via a provided callback.

#### 2. Command Execution Mechanism

To execute commands inside the created Pod:
- **Primary Method (K8s Exec API)**: Uses Kubernetes Pod `exec` WebSocket/SPDY streams (`client.CoreV1Api().connect_get_namespaced_pod_exec`). This avoids requiring custom agents or open network ports inside the sandbox Pod.
- **File Transfer**: Uses `tar` streams over K8s `exec` streams to seamlessly copy files into (`upload_file`) and out of (`download_file`) the Pod.

#### 3. Lifecycle Management & Garbage Collection

To ensure Pods are reliably cleaned up and do not become orphaned:
- **Active Deadline**: Pods are configured with `activeDeadlineSeconds` matching the specified sandbox TTL.
- **Context Cleaners**: `sb.terminate()` or Python `__exit__` cleanly calls `delete_namespaced_pod`.
- **K8s Owner References**: When deployed via KubeRay, sandbox Pods will set the `ownerReferences` pointing to the parent `RayCluster` or `RayJob` CRD so Kubernetes automatically cascades deletions if the cluster terminates.

---

### Logical Resourcing for Sandboxes

Ray will provide unified resource scheduiling and allocation for sandboxes by leveraging Ray's logical resource management and scheduling system. When users request resources for a sandbox, Ray will reserve those resources on the node where the sandbox is created, preventing those resources from being allocated to other tasks or actors.

Resource scheduling will be implemented differently depending on the sandbox backend:
- **Node-Local Backends (e.g., `gvisor`)**: For backends that execute sandbox environments directly on the host worker node, requesting resources during sandbox creation (e.g., `resources={"cpu": "1000m", "memory": "512Mi"}`) reserves corresponding logical resources (e.g., 1 CPU) on that Ray worker node. These logical resources remain reserved in Ray's scheduler for the duration of the sandbox's lifecycle and are released back to the node when `sb.terminate()` or context manager exit occurs.
- **Remote / Cluster-Managed Backends (e.g., `kubernetes`)**: For backends where sandboxes run off-node on external compute infrastructure (such as Kubernetes Pods scheduled by the Kubernetes control plane), the sandbox does not consume host Raylet logical resources, leaving resource allocation and scheduling accounting to the remote cluster manager.

---

## Compatibility, Deprecation, and Migration Plan

- **Backwards Compatibility**: This change is 100% backwards compatible. `ray.experimental.sandbox` is a completely new, independent library. No existing Ray Core APIs, Raylet behaviors, or `@ray.remote` actor options are modified or deprecated.
- **Migration Path**: Frameworks currently using custom subprocess wrappers or external REST APIs can directly replace their execution code with `ray.experimental.sandbox.create(runtime="gvisor", ...)` or `ray.experimental.sandbox.create(runtime="kubernetes", ...)` calls.

---

## Test Plan and Acceptance Criteria

### Unit Tests
- `ray.experimental.sandbox` API tests (mocking Kubernetes API and `runsc` CLI invocations).
- `GVisorSandboxEnv` OCI spec generation and CLI command string verification (`runsc create`, `start`, `exec`, `kill`, `delete`).
- `SandboxPool` concurrency, allocation, and release tests.
- Configuration validation (invalid resource specs, timeouts, image names).

### Integration & E2E Tests
- **Local K8s Testing**: E2E tests using Kind/Minikube to verify Pod creation, command execution, stdout/stderr capture, file upload/download, and termination.
- **gVisor Runtime Testing**: E2E tests on nodes with `runsc` installed:
  - **Startup Latency**: Benchmark sandbox instantiation to verify sub-100ms startup times.
  - **Bin Packing Density**: Verify execution of 100+ concurrent gVisor sandboxes on a single worker node.
  - **Command Execution & Files**: Test `runsc exec`, file upload/download, stdout/stderr capture, and timeout enforcement.
- **Security Validation**: Verify that non-root, read-only rootfs, and dropped capability settings prevent privilege escalation inside both Kubernetes Pods and gVisor sandboxes.
- **Orphan Sandbox Cleanup**: Test process crashes (killing driver script or worker process) and verify that orphan K8s Pods and `runsc` sandbox containers are cleaned up cleanly.

### Acceptance Criteria
1. Complete `ray.experimental.sandbox` Python package exported under `ray.experimental.sandbox`.
2. Fully functional `kubernetes` and `gvisor` sandbox implementations supporting sandbox lifecycle, command execution, and file transfers.
3. `SandboxPool` implementation for high-throughput parallel execution across both runtimes.
4. Comprehensive documentation and example script showing RL code evaluation using `ray.experimental.sandbox`.

---

## (Optional) Follow-on Work

- **Additional Runtimes**: Support for Firecracker microVMs (`runtime="firecracker"`), other open-source projects for sandboxes (agent-sandbox, skypilot sandboxing, huggingface openenv, etc.), and third-party APIs (modal, etc.).
- **gVisor Checkpoint / Restore**: Fast-start warm sandbox pools leveraging `runsc checkpoint` and `runsc restore` to snapshot pre-warmed Python interpreter states for sub-10ms execution start.
- **Pre-warmed Pod Pools**: Fast-start warm pod pools with instant snapshotting/restore for Kubernetes sandboxes.
