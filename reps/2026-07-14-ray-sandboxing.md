# Summary

### General Motivation

Ray is emerging as the de facto orchestrator for Reinforcement Learning (RL) frameworks and AI agent workflows (e.g., veRL, SLIME, NeMo-RL, DeepSeek-R1 style training, coding benchmarks, and tool-use evaluation). A key requirement in these workloads is the safe execution of untrusted, model-generated code and tool calls during training and evaluation loops.

Currently, running untrusted code directly on Ray worker nodes poses severe security and operational risks, including arbitrary host access, data leakage, and cluster instability. To avoid these risks, frameworks often delegate execution to external sandbox services or third-party APIs. However, this externalized approach fragments the Ray ecosystem with ad-hoc abstractions and increases orchestration complexity.

This proposal addresses these challenges by introducing an (experimental) Ray Sandbox library to programmatically manage isolated sandbox environments. The `ray.experimental.sandbox` library will provide a common denominator abstraction for secure sandboxing with unified resource scheduling in Ray. The initial implementation will focus solely on a gVisor-based backend (`runsc`) operating directly on Ray worker nodes for fast sandbox startup, strong kernel isolation, and dense bin packing of execution environments.

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
                                  |
                                  v
+-------------------------------------------------------------------+
|                         GVisorSandboxEnv                          |
|            (Manages host `runsc` spec & lifecycle)                 |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                        Ray Worker Node                            |
|   +-----------------------+       +-----------------------+       |
|   | gVisor Sandbox 1      |       | gVisor Sandbox 2      |       |
|   | (Isolated Execution)  |       | (Isolated Execution)  |       |
|   +-----------------------+       +-----------------------+       |
+-------------------------------------------------------------------+
```

### Python API Design

The `ray.experimental.sandbox` library will provide an intuitive, high-level abstraction for creating and managing isolated sandbox environments. The initial implementation focuses solely on gVisor (`runsc`), providing node-local container sandboxing optimized for fast startup times (sub-100ms) and high-density bin packing of concurrent execution tasks.

#### Example: Basic Sandbox Creation and Command Execution

```python
import ray
from ray import sandbox

ray.init()

# Create a gVisor sandbox environment on the local worker node
sb = sandbox.create(
    cpu=1.0,
    memory="512Mi",
    timeout=300,  # Auto-terminate after 5 minutes
)

# Execute code or bash commands inside the sandbox
result = sb.exec("python -c 'import sys; print(sys.version)'")

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

with sandbox.create() as sb:
    # Upload local files or scripts into the gVisor sandbox
    sb.upload_file(local_path="model_eval.py", remote_path="/tmp/model_eval.py")

    # Run execution
    res = sb.exec("python /tmp/model_eval.py --input /tmp/data.json")

    # Download output artifacts
    sb.download_file(remote_path="/tmp/output.json", local_path="results.json")
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
    def exec(self, instance_id: str, command: str, timeout: int = None, env: dict = None) -> ExecutionResult:
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

### Sandboxing with gVisor

The `gvisor` sandbox environment implementation (`GVisorSandboxEnv`) uses gVisor's OCI runtime (`runsc`) to execute untrusted code in lightweight, kernel-isolated sandboxes directly on Ray worker nodes.

#### Motivation: Fast Startup & Dense Bin Packing

In RL training and LLM agent evaluation loops, workers execute thousands of short-lived code snippets per second. Traditional Kubernetes Pod creation adds multi-second latency (scheduling, API server overhead, networking setup), creating a bottleneck for high-throughput rollout pipelines.

gVisor addresses these challenges through two key advantages:
1. **Fast Startup**: By intercepting application system calls in user space via the gVisor application kernel, sandbox containers start in tens of milliseconds (sub-100ms latency), bypassing container daemon and Kubernetes control plane overhead.
2. **Dense Bin Packing**: gVisor sandboxes have a minimal memory footprint and near-zero idle CPU overhead per instance. Ray worker nodes can densely pack hundreds or thousands of concurrent gVisor sandboxes alongside Ray actors and tasks without requiring separate VM instances or cluster nodes.

#### 1. Sandbox Provisioning and OCI Spec Generation

When `sandbox.create(...)` is called on a Ray worker node, `GVisorSandboxEnv` performs the following steps:
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
- **`exec(instance_id, command, timeout, env)`**:
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

### Logical Resourcing for Sandboxes

Ray will provide unified resource scheduling and allocation for sandboxes by leveraging Ray's logical resource management and scheduling system. When users request resources during sandbox creation (e.g., `cpu=1.0, memory="512Mi"`), Ray will reserve those logical resources on the host worker node where the gVisor sandbox is created, preventing those resources from being allocated to other tasks or actors.

These logical resources remain reserved in Ray's scheduler for the duration of the sandbox's lifecycle and are automatically released back to the worker node when `sb.terminate()` or context manager exit occurs.

---

## Compatibility, Deprecation, and Migration Plan

- **Backwards Compatibility**: This change is 100% backwards compatible. `ray.experimental.sandbox` is a completely new, independent library. No existing Ray Core APIs, Raylet behaviors, or `@ray.remote` actor options are modified or deprecated.
- **Migration Path**: Frameworks currently using custom subprocess wrappers or external REST APIs can directly replace their execution code with `ray.experimental.sandbox.create(...)` calls.

---

## Test Plan and Acceptance Criteria

### Unit Tests
- `ray.experimental.sandbox` API tests (mocking `runsc` CLI invocations).
- `GVisorSandboxEnv` OCI spec generation and CLI command string verification (`runsc create`, `start`, `exec`, `kill`, `delete`).
- Configuration validation (invalid resource specs, timeouts, image names).

### Integration & E2E Tests
- **gVisor Runtime Testing**: E2E tests on nodes with `runsc` installed:
  - **Startup Latency**: Benchmark sandbox instantiation to verify sub-100ms startup times.
  - **Bin Packing Density**: Verify execution of 100+ concurrent gVisor sandboxes on a single worker node.
  - **Command Execution & Files**: Test `runsc exec`, file upload/download, stdout/stderr capture, and timeout enforcement.
- **Security Validation**: Verify that non-root, read-only rootfs, and dropped capability settings prevent privilege escalation inside gVisor sandboxes.
- **Orphan Sandbox Cleanup**: Test process crashes (killing driver script or worker process) and verify that orphan `runsc` sandbox containers are cleaned up cleanly.

### Acceptance Criteria
1. Complete `ray.experimental.sandbox` Python package exported under `ray.experimental.sandbox`.
2. Fully functional `gvisor` sandbox implementation supporting sandbox lifecycle, command execution, and file transfers.
3. Comprehensive documentation and example script showing RL code evaluation using `ray.experimental.sandbox`.

---

## (Optional) Follow-on Work

- **Additional Runtimes**: Support for Kubernetes Pod backend (`runtime="kubernetes"`), Firecracker microVMs (`runtime="firecracker"`), other open-source projects for sandboxes (agent-sandbox, skypilot sandboxing, huggingface openenv, etc.), and third-party APIs (modal, etc.).
- **gVisor Checkpoint / Restore**: Fast-start warm sandbox pools leveraging `runsc checkpoint` and `runsc restore` to snapshot pre-warmed Python interpreter states for sub-10ms execution start.
