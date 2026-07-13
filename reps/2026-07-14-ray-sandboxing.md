# Summary

### General Motivation

Ray is emerging as the de facto orchestrator for Reinforcement Learning (RL) frameworks and AI agent workflows (e.g., veRL, SLIME, NeMo-RL, DeepSeek-R1 style training, coding benchmarks, and tool-use evaluation). A key requirement in these workloads is the safe execution of untrusted, model-generated code and tool calls during training and evaluation loops.

Currently, running untrusted code directly on Ray worker nodes poses severe security and operational risks, including arbitrary host access, data leakage, and cluster instability. To avoid these risks, frameworks often delegate execution to external sandbox services or third-party APIs. However, this externalized approach fragments the Ray ecosystem with ad-hoc abstractions and increases orchestration complexity.

This proposal addresses these challenges by introducing a Ray Sandbox library to programmatically manage isolated sandbox environments. Following industry best practices, `ray.sandbox` will provide a common denominator abstraction with support for widely used backends (e.g., Kubernetes, container runtimes, and microVMs). The initial implementation will support a Kubernetes-based sandbox envirionments, using Pods to execute code in isolated environments.

### Should this change be within `ray` or outside?

Within `ray` as a library (`ray.sandbox`).

## Stewardship

### Required Reviewers

@MengjinYan
@richardliaw
@kouroshHakha

### Shepherd of the Proposal (should be a senior committer)

@pcmoritz

## Design and Architecture

### Overview

`ray.sandbox` introduces an abstraction layer between Ray workloads (tasks, actors, or driver scripts) and underlying container or micro-VM orchestration engines.

```
+-------------------------------------------------------------------+
|               Ray Application / RL Framework                       |
|           (e.g., veRL, SLIME, Rollout Workers, Agents)            |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                            ray.sandbox                            |
|  +---------------------+  +-----------------+  +-----------------+|
|  |   Sandbox           |  |  SandboxPool    |  |  ExecResult     ||
|  +---------------------+  +-----------------+  +-----------------+|
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                          SandboxEnv Interface                     |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                       KubernetesSandboxEnv                        |
|            (Manages K8s Pods, SecurityContext, TTL)               |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                         Kubernetes Cluster                        |
|   +-----------------------+       +-----------------------+       |
|   | Sandbox Pod 1         |       | Sandbox Pod 2         |       |
|   | (Isolated Execution)  |       | (Isolated Execution)  |       |
|   +-----------------------+       +-----------------------+       |
+-------------------------------------------------------------------+
```

### Python API Design

The `ray.sandbox` library will provide an intuitive, high-level abstractions for creating and managing isolated sandbox environments across common execution backends.

The initial implementation introduces support for Kubernetes Pods, incorporating industry best practices for security and performance—such as Pod security hardening and pre-warmed sandbox pool management for high-throughput RL evaluation workloads.

#### Example: Basic Sandbox Creation and Command Execution

```python
import ray
from ray import sandbox

ray.init()

# Create a Kubernetes sandbox environment
sb = sandbox.create(
    runtime="kubernetes",
    image="python:3.10-slim",
    resources={"cpu": "1000m", "memory": "2Gi"},
    namespace="ray-sandboxes",
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

with sandbox.create(runtime="kubernetes", image="python:3.10-slim") as sb:
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

### Kubernetes Env

The first sandbox environment implementation will be `kubernetes`, which will support creating dedicated Kubernetes Pods to serve as isolated sandbox environments.
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
- Apply any user defined modfications to the pod spec via a provided callback.

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

## Compatibility, Deprecation, and Migration Plan

- **Backwards Compatibility**: This change is 100% backwards compatible. `ray.sandbox` is a completely new, independent library. No existing Ray Core APIs, Raylet behaviors, or `@ray.remote` actor options are modified or deprecated.
- **Migration Path**: Frameworks currently using custom subprocess wrappers or external REST APIs can directly replace their execution code with `ray.sandbox.create(env="kubernetes", ...)` calls.

---

## Test Plan and Acceptance Criteria

### Unit Tests
- `ray.sandbox` API tests (mocking Kubernetes API calls).
- `SandboxPool` concurrency, allocation, and release tests.
- Configuration validation (invalid resource specs, timeouts, image names).

### Integration & E2E Tests
- **Local K8s Testing**: E2E tests using Kind/Minikube to verify Pod creation, command execution, stdout/stderr capture, file upload/download, and termination.
- **Security Validation**: Verify that non-root, read-only rootfs, and dropped capability settings prevent privilege escalation inside the Pod.
- **Orphan Pod Cleanup**: Test process crashes (killing driver script) and verify that K8s activeDeadline cleans up orphaned sandbox Pods within expected timeout limits.

### Acceptance Criteria
1. Complete `ray.sandbox` Python package exported under `ray.sandbox`.
2. Fully functional `kubernetes` sandbox implementation supporting Pod lifecycle, command execution, and file transfers.
3. `SandboxPool` implementation for high-throughput parallel execution.
4. Comprehensive documentation and example script showing RL code evaluation using `ray.sandbox`.

---

## (Optional) Follow-on Work

- **Additional Runtimes**: Support for gVisor standalone (`runtime="gvisor"`), Firecracker microVMs (`runtime="firecracker"`), other open-source projects for sandboxes (agent-sandbox, skypilot sandboxing, huggingface openenv, etc) and third party APIs (modal, etc).
- **Pre-warmed Pod Pools**: Fast-start warm pod pools with instant snapshotting/restore to reduce Pod start latency below 100ms.
