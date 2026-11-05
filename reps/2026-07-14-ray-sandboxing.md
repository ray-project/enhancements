# Summary

### General Motivation

Ray is emerging as the de facto orchestrator for Reinforcement Learning (RL) frameworks (e.g. veRL, MILES, SkyRL, SLIME, NeMo-RL, etc). A key requirement in these workloads is the safe execution of untrusted, model-generated code and tool calls during training and evaluation loops.

Currently, running untrusted code directly on Ray worker nodes poses severe security and operational risks, including arbitrary host access, data leakage, and cluster instability. To mitigate these risks, frameworks often delegate execution to external sandbox services or third-party APIs. These services play an important role in the ecosystem and will continue to be fully supported. That said, we believe there is also value in a native sandbox solution that works out of the box on any Ray cluster and provides strong isolation for agentic workloads.

This proposal addresses these challenges by introducing an (experimental) Ray Sandbox library. The library will provide common building blocks for secure sandboxing with unified resource scheduling in Ray. The initial implementation will focus solely on a gVisor-based backend (`runsc`) operating directly on Ray worker nodes for fast sandbox startup, strong kernel isolation, and dense bin packing of execution environments.

### Should this change be within `ray` or outside?

Within `ray` as an experimental library (`ray.experimental.sandbox`).

## Stewardship

### Required Reviewers

@edoakes
@MengjinYan
@richardliaw
@kouroshHakha

### Shepherd of the Proposal (should be a senior committer)

@robertnishihara @pcmoritz

## Design and Architecture

### Overview

Ray will introduce the following experimental APIs to provide building blocks for sandboxing within Ray:
1. A `SandboxRuntime` class (in `ray.experimental.sandbox.runtime`) - this is a common interface for managing local sandbox environments. For the initial version, we will only support gVisor as the sandbox runtime. `SandboxRuntime` is a concrete class over a backend (`BaseSandboxBackend`); `GVisorSandboxBackend` is the gVisor one. One `SandboxRuntime` can host many sandboxes, and keep some booted ahead of time in a warm pool (see [Design updates](#design-updates-november-2026)).
2. A `Sandbox` Ray actor (in `ray.experimental.sandbox`) that wraps `SandboxRuntime` and provides a high-level API for creating and managing sandbox environments. The main purpose of this API is to manage scheduling and lifecycle of sandboxes using familiar Ray Actor API.
3. A Modal-like library (in `ray.experimental.sandbox`) that manages `ray.experimental.sandbox.Sandbox` actors.
4. An HTTP API and a gRPC facade that speaks the Modal SDK's wire protocol (in `ray.experimental.sandbox.http`), served with Ray Serve. Each API sandbox is a named, detached `SandboxHost` actor, so any Serve replica can reach it and the service keeps no state of its own.

> [!WARNING]
> All experimental APIs are subject to review prior to GA graduation and may be modified or removed.

```
+-------------------------------------------------------------------+
|               Ray Application / RL Framework                       |
|           (e.g., veRL, SLIME, Rollout Workers, Agents)            |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                      ray.experimental.sandbox                     |
|                   (High-Level Modal-Like Library)                 |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|               ray.experimental.sandbox.Sandbox                    |
|             (Ray Actor wrapping `SandboxRuntime`)                 |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                   ray.experimental.sandbox.runtime                |
|                      SandboxRuntime Interface                     |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                        GVisorSandboxBackend                       |
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

### API Design

#### ray.experimental.sandbox.runtime.SandboxRuntime

This is the lowest-level API for sandboxing which provides fine-grained control over the sandbox environments.

Below is the interface for `SandboxRuntime`. Ray will only support a gVisor backend for the initial release.
As built, `create` takes the `SandboxConfig` fields as keyword arguments, `exec` takes an argv list or a shell string,
and the class also has `write_file` / `read_file`, `get_status`, `set_egress_allowlist`, `pause` / `resume`,
`checkpoint` / `restore` (see the [checkpoint and restore REP](https://github.com/ray-project/enhancements/pull/77)), and `terminate`. Its constructor takes warm-pool
profiles and a warm-pool store, described under [Design updates](#design-updates-november-2026).
```python
class SandboxRuntime(ABC):
    """Low-level interface for managing local sandbox runtime environments."""

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
        """Copy file from the sandbox to local."""
        pass

    @abstractmethod
    def delete(self, instance_id: str) -> None:
        """Clean up and terminate the sandbox instance."""
        pass
```

#### ray.experimental.sandbox.Sandbox

`ray.experimental.sandbox.Sandbox` is a Ray actor that wraps `SandboxRuntime` and provides a high-level API for creating and managing sandbox environments. The main purpose of this actor is to manage scheduling and lifecycle of sandboxes using familiar Ray Core APIs.

Below is the interface for the `Sandbox` actor:
```python
@ray.remote
class Sandbox():
    """Ray actor interface for managing scheduling and lifecycle of an isolated sandbox."""

    def __init__(self, runtime: SandboxRuntime):
        self.runtime = runtime

    def exec(self, command: str, timeout: int = None, env: dict = None) -> ExecutionResult:
        """Execute a command inside the sandbox."""
        pass

    def upload_file(self, local_path: str, remote_path: str) -> None:
        """Copy local file into the sandbox."""
        pass

    def download_file(self, remote_path: str, local_path: str) -> None:
        """Copy file from the sandbox to local."""
        pass

    def delete(self) -> None:
        """Clean up and terminate the sandbox instance."""
        pass
```


#### ray.experimental.sandbox

The `ray.experimental.sandbox` library will provide a high-level abstraction for creating and managing isolated sandbox environments.
Under the hood, it will use a `ray.experimental.sandbox.Sandbox` actor for scheduling and resource assignment and `SandboxRuntime` to manage the sandbox runtime
using gVisor. For the initial version of this API, we will provide a simple `create` and `create_async` API that returns a `SandboxHandle`. Below is the interface:

```python
def create(
    config: Optional[SandboxConfig] = None,
    **kwargs,
) -> SandboxHandle:
    """Create a sandbox environment."""
    pass


def create_async(
    config: Optional[SandboxConfig] = None,
    **kwargs,
) -> "ray.util.async_func[SandboxHandle]":
    """Create a sandbox environment asynchronously."""
    pass


class SandboxHandle():
    """High-level handle interface for interacting with a Ray sandbox."""

    def exec(self, command: str, timeout: int = None, env: dict = None) -> ExecutionResult:
        """Execute a command inside the sandbox."""
        pass

    def upload_file(self, local_path: str, remote_path: str) -> None:
        """Copy local file into the sandbox."""
        pass

    def download_file(self, remote_path: str, local_path: str) -> None:
        """Copy file from the sandbox to local."""
        pass

    def delete(self) -> None:
        """Clean up and terminate the sandbox instance."""
        pass
```

### Examples

#### Example: Basic Sandbox Creation and Command Execution

```python
import ray
from ray.experimental import sandbox

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
from ray.experimental import sandbox

with sandbox.create() as sb:
    # Upload local files or scripts into the gVisor sandbox
    sb.upload_file(local_path="model_eval.py", remote_path="/tmp/model_eval.py")

    # Run execution
    res = sb.exec("python /tmp/model_eval.py --input /tmp/data.json")

    # Download output artifacts
    sb.download_file(remote_path="/tmp/output.json", local_path="results.json")
```


#### Example: Create a `ray.experimental.sandbox.Sandbox` actor

```python
from ray.experimental.sandbox import Sandbox

sandbox = Sandbox.options(
    num_cpus=1,
    memory=1e9
).remote(
    timeout=1800
)

sandbox.write_file.remote("/workspace/main.py", "print('Hello from Ray Sandbox!')")
result = ray.get(sandbox_client.exec.remote("python3 /workspace/main.py"))
```

#### Example: Create a custom actor using ray.experimental.SandboxRuntime

```python
import ray
from ray.experimental.sandbox.runtime import SandboxRuntime

@ray.remote
class CustomSandbox():
    def __init__(self):
      self.runtime = SandboxRuntime()
      self.instance_id = None

    def create(self, config: SandboxConfig) -> str:
        """Provision the sandbox instance and return unique instance ID."""
        # custom create logic using self.runtime.create()
        pass

    def exec(self, instance_id: str, command: str, timeout: int = None, env: dict = None) -> ExecutionResult:
        """Execute a command inside the specified sandbox."""
        # custom exec logic using self.runtime.exec()
        pass

sandbox = CustomSandbox.remote()
result = ray.get(sandbox.exec.remote("echo 'Hello from Ray Sandbox!'"))
```

---

### Sandboxing with gVisor

The `gvisor` sandbox environment implementation (`GVisorSandboxRuntime`) uses gVisor's OCI runtime (`runsc`) to execute untrusted code in lightweight, kernel-isolated sandboxes directly on Ray worker nodes.

#### Motivation: Fast Startup & Dense Bin Packing

In RL training and LLM agent evaluation loops, workers execute thousands of short-lived code snippets per second. Traditional Kubernetes Pod creation adds multi-second latency (scheduling, API server overhead, networking setup), creating a bottleneck for high-throughput rollout pipelines.

gVisor addresses these challenges through two key advantages:
1. **Fast Startup**: By intercepting application system calls in user space via the gVisor application kernel, sandbox containers start in tens of milliseconds (sub-100ms latency), bypassing container daemon and Kubernetes control plane overhead.
2. **Dense Bin Packing**: gVisor sandboxes have a minimal memory footprint and near-zero idle CPU overhead per instance. Ray worker nodes can densely pack hundreds or thousands of concurrent gVisor sandboxes alongside Ray actors and tasks without requiring separate VM instances or cluster nodes.

Since gVisor is a user-space Linux kernel implementation, it can provide sandboxes for any Linux-based environment. While it may require additional pod security configurations on Kubernetes, these rely on standard features supported across most Kubernetes distributions. Any future runtimes must meet the same criteria.

#### 1. Sandbox Provisioning and OCI Spec Generation

When `SandboxRuntime.create(...)` is called on a Ray worker node, `GVisorSandboxBackend` performs the following steps:
- **OCI Bundle Directory**: Creates a unique OCI bundle directory under `/tmp/ray/sandbox/<instance_id>/` (IDs are `ray-sandbox-<12 hex digits>`) containing an OCI specification `config.json` and the bundle's owner file. The root filesystem is the image's cached EROFS file (see [Container Image Support](#6-container-image-support)), which gVisor mounts inside its own kernel, or a kernel overlayfs over it when the spec needs one (OCI hooks, such as a GPU's CDI hooks, or mount points the image lacks).
- **OCI Config Generation**: Generates `config.json` (or calls `runsc spec` with standard overrides):
  - **Process Config**: Sets entrypoint process, working directory, UID/GID, and environment variables.
  - **Resource Limits**: CPU and memory limits are enforced by Ray itself, in a child cgroup per sandbox under a cgroup v2 subtree that node provisioning delegates (`RAY_SANDBOX_CGROUP_PATH`), and they fail closed: a create with limits fails on a node that can't enforce them. A sandbox with a CPU limit is also pinned to CPUs chosen under a node-wide lock, so concurrent creates in any process pick apart.
  - **Security Hardening**: Enforces rootless execution (`runAsNonRoot`), read-only rootfs (`readonly: true`), drops capabilities (`capabilities.bounding: []`), and enables seccomp syscall filtering.
- **Sandbox Creation Calls**: one `runsc --root=/tmp/runsc run --bundle <bundle_dir> <instance_id>` launch process per sandbox, started in the owning worker's process group (wrapped in the sandbox's cgroup, and in a private user and network namespace for `network="public"`). The create returns once runsc reports the container running: today by polling `runsc state`, and with ray-project/ray#67074 by waiting for runsc's pid file.

#### 2. Command Execution Mechanism (`runsc exec`)

To execute commands inside an active gVisor sandbox instance:
- **`exec(instance_id, command, timeout, env)`**:
  - Calls `runsc --root=/tmp/runsc exec -cwd <cwd> -internal-pid-file <pid_file> [-user <user>] [-env K=V ...] <instance_id> <argv>` (a string command runs under the sandbox's `shell -c`)
  - Connects standard IO streams to capture stdout and stderr directly from the sandbox process.
  - Enforces process timeouts by killing the command and every process it started inside the sandbox, found through the pid file, if command execution exceeds the timeout limit.
  - Returns `ExecutionResult` containing `exit_code`, `stdout`, `stderr`, and precise execution `duration_ms`.

#### 3. File Transfer Mechanism

The root filesystem is an immutable EROFS image that gVisor mounts inside its own kernel, so the host can't write into a running sandbox's files directly. File operations go through `runsc exec` instead:
- **`write_file` / `upload_file`**: streams the bytes into a `cat > <path>` run inside the sandbox.
- **`read_file` / `download_file`**: runs `cat -- <path>` inside the sandbox and returns, or writes, its output.

#### 4. Lifecycle Management & Garbage Collection

- **Termination**: `sb.terminate()` or context manager exit executes:
  1. `runsc --root=<root_dir> kill <instance_id> SIGKILL`
  2. `runsc --root=<root_dir> delete <instance_id>`
  3. Removes the temporary OCI bundle directory `<bundle_dir>`.
- **Owner Death**: A sandbox doesn't outlive the worker that owns it. Its launch process stays in the worker's process group, which the Raylet kills when the worker exits or crashes, and runsc's sandbox dies with it through a parent-death signal. What it leaves on disk (runsc state, the bundle with its writable overlay, the image pin) is reclaimed by the next create on the node, in the background and at most once a minute per process: every bundle's owner file says whether its owner is still alive (ray-project/ray#67022, and since ray-project/ray#67731 through one lock file per owning process).

#### 5. Release / Dependency Updates

Supporting gVisor will require bundling `runsc` into Ray container images. The `runsc` binary is ~30MB, and current gVisor releases ship it with sidecar binaries in `gvisor-bin/` that must sit next to it. Workers also need `mkfs.erofs` (erofs-utils 1.7 or later) for the image cache, `slirp4netns` and `nft` for `network="public"`, and util-linux tools (plus `erofsfuse` on workers without mount privileges) for overlayfs root filesystems. On nodes whose kernel uses 64 KiB pages, such as NVIDIA Vera, `runsc` and erofs-utils must be built for that page size.

#### 6. Container Image Support

The initial version of the gVisor runtime did not support specifying container images. Container images are now supported:
* `create(image=...)` pulls the image from its registry (with the node's Docker credentials, and an optional Docker Hub mirror, `RAY_SANDBOX_REGISTRY_MIRROR`) into the node's image cache under `/tmp/ray/sandbox/images`.
* The cache keeps each image as one EROFS file built with `mkfs.erofs --tar`, which every sandbox of that image shares read-only. gVisor mounts it inside its own kernel, so files keep the image's owners and `chown` works for any uid without privileges or id mappings on the node.
* Each sandbox pins the images it uses, and the cache evicts unpinned images when the node runs short of disk.

---

## Design updates (November 2026)

> [!NOTE]
> This section records what changed in the design after this REP was accepted, and why. The changes come from running Ray Sandbox at scale (a 352-CPU NVIDIA Vera node with 64 KiB pages, and CoreWeave and EKS clusters) and are in the PRs linked below; PRs marked open are not merged yet. Checkpoint and restore are covered by their own REP ([ray-project/enhancements#77](https://github.com/ray-project/enhancements/pull/77)).

| Area | Change | PR | Status |
| --- | --- | --- | --- |
| Warm pools | `SandboxRuntime` keeps fixed-size pools of booted sandboxes, refilled on delete | [ray#67077](https://github.com/ray-project/ray/pull/67077) | open |
| Warm pool store | Pools publish to a store (Ray actor or any database), so callers on any actor find ready sandboxes | [ray#67728](https://github.com/ray-project/ray/pull/67728) | open |
| Multi-sandbox actors | One owner lock per process instead of two open files per sandbox | [ray#67731](https://github.com/ray-project/ray/pull/67731) | open |
| Ownership and reclaim | A sandbox stops with its owner; the next create reclaims what it left | [ray#67022](https://github.com/ray-project/ray/pull/67022) | merged |
| Resource limits | CPU and memory enforced with host cgroups, failing closed | [ray#66932](https://github.com/ray-project/ray/pull/66932) | merged |
| Resource limits | Whole physical cores per sandbox; a node-wide cap on each sandbox's host tasks | [ray#67747](https://github.com/ray-project/ray/pull/67747), [ray#67748](https://github.com/ray-project/ray/pull/67748) | open |
| Image cache | Build overlapping image pulls in child interpreters | [ray#67746](https://github.com/ray-project/ray/pull/67746) | open |
| runsc's root directory | Remove the lock files deletes leave behind | [ray#67729](https://github.com/ray-project/ray/pull/67729), [gvisor#16903](https://github.com/google/gvisor/pull/16903), [gvisor#16902](https://github.com/google/gvisor/pull/16902) | open |
| Start-up path | Pid-file readiness, `GOMAXPROCS=2` for runsc calls, less work per create | [ray#67074](https://github.com/ray-project/ray/pull/67074), [ray#67075](https://github.com/ray-project/ray/pull/67075), [ray#67076](https://github.com/ray-project/ray/pull/67076) | open |
| Bursts | Switch roots without pivot_root(2) on Linux 7.1+ | [gvisor#16901](https://github.com/google/gvisor/pull/16901) | open |
| Exec path | Don't join a sandbox's cgroup on every exec | [ray#67690](https://github.com/ray-project/ray/pull/67690) | open |
| gRPC facade | Exec state lives on the sandbox's host actor | [ray#67078](https://github.com/ray-project/ray/pull/67078) | merged |
| Node settings | Default resolvers and extra runsc flags per node | [ray#67691](https://github.com/ray-project/ray/pull/67691), [ray#67692](https://github.com/ray-project/ray/pull/67692) | open |
| arm64 and 64 KiB pages | Start apps with SSBS set; put vDSO parameters where the vDSO reads them | [gvisor#16814](https://github.com/google/gvisor/pull/16814), [gvisor#16815](https://github.com/google/gvisor/pull/16815) | open |
| arm64 | Complete trapped WFE/WFI and emulate ID register reads on KVM; show apps BF16, I8MM and other stateless HWCAP2 features | [gvisor#16909](https://github.com/google/gvisor/pull/16909), [gvisor#16910](https://github.com/google/gvisor/pull/16910) | open |
| Sandbox init | PID 1 reaps orphaned processes | [ray#67749](https://github.com/ray-project/ray/pull/67749) | open |
| Tests | An out-of-memory kill stays inside its sandbox | [ray#67694](https://github.com/ray-project/ray/pull/67694) | open |

### Warm pools in `SandboxRuntime`

Creating a sandbox boots gVisor, which takes a fraction of a second, and much longer when many sandboxes start at once. A caller that knows the configurations it will create passes them to `SandboxRuntime(warm_pool=[...])` as profiles: `create()` keyword arguments plus a `size`. The runtime boots each profile's sandboxes ahead of time, and a create whose boot-time fields match a profile claims one at once.

- **Boot key.** Everything fixed when a sandbox boots must match: image, CPU, memory, disk, workdir, network, DNS, egress allowlist, capabilities, shell, GPUs, mounts, `readonly` and `rootless`. A create's `env`, `ttl_seconds` and `timeout_seconds` need not: the backend's `assign_config` gives the claimed sandbox the create's config, so every command runs with the create's `env`, its TTL starts at the create, and `get_config()` and checkpoints see it. A create with a custom OCI spec transform never takes a pooled sandbox.
- **Fixed-size slots, refilled on delete.** Each profile has `size` slots. A slot is empty, booting, ready, claimed (handed to a create) or deleting (its sandbox died while ready). A claimed sandbox keeps its slot until it is deleted, by `delete`, its TTL, or a checkpoint that stops it, and only then does a new sandbox boot in that slot. So a pool never holds more than `size` sandboxes, and a create that finds none ready boots cold, outside the pool. Refilling on claim instead would let a pool grow without bound under churn.
- **Boots and failures.** Boots run four at a time. A profile whose boot fails retries after 30 s. A ready sandbox whose launch process has exited is discarded and its slot rebooted rather than handed out.
- **Close.** `close()` deletes the ready sandboxes; claimed ones stay with their creates, and deleting them later boots nothing.

On the Vera node, a pool of 500 handed 500 simultaneous creates (from 64 threads) their sandboxes at a create p50 of 0.13 ms (p99 4.6 ms). The burst finished in 1.1 s, and a first command ran 0.14 s (p50) after its create. Cold, the same burst takes 34 s. After the 500 were deleted, the pool refilled its slots in 39 s and never held more than 500.

### Warm pool store (a "permit DB")

A pool belongs to the runtime, and so the actor, that boots it, and only that runtime's creates can take its sandboxes. With pools on many actors, a caller can't tell which actor has a sandbox ready. A `WarmPoolStore` is where pools say so:

```python
class WarmPoolStore(ABC):
    def publish(self, host: str, entries: List[WarmPoolEntry]) -> None:
        """Replace everything known about `host` with `entries`; [] removes it."""

    def ready(self, profile: str) -> List[WarmPoolEntry]:
        """The ready sandboxes of `profile`, on every host."""

@dataclass(frozen=True)
class WarmPoolEntry:
    host: str          # the runtime holding it, by the name callers reach it by
    profile: str       # profile name (default: its image)
    slot: int          # 0 .. size-1; kept while new sandboxes boot in the slot
    sandbox_id: str
    state: str         # "ready" or "claimed"
    node_id: Optional[str]
    since: float
    expires_at: float
```

`SandboxRuntime(warm_pool=..., warm_pool_store=store)` publishes every sandbox its pool holds, ready or claimed, whenever that changes (coalesced, at most every 50 ms) and every 10 s. Entries expire 30 s after their host's last publish, and closing the pool removes them. A caller lists `store.ready(profile)` and sends its create to a host that has one. `RayWarmPoolStore` keeps the records in a detached actor (no CPU, restarted forever), which the hosts' next publishes refill if it restarts. Any database can implement the two methods instead. Because a slot keeps its `(host, profile, slot)` while new sandboxes boot in it, each slot can be a row in a permits table, and the table is the cluster's warm capacity.

**The store is an index, not the claim authority.** Only a host hands out its sandboxes, and only ones still ready on it, so a stale record costs a cold boot and never hands one sandbox to two creates. Stores need no transactions, leases or claim tokens, and correctness doesn't depend on any store implementation. We rejected store-side atomic claims (claim tokens, leases, and an `adopt()` for the host): they add API, need leak handling for callers that crash holding a claim, and make correctness depend on every store. A pool keeps its host name until its last publish is done, so a slow store can't let a closed pool's late publish replace the records of a new pool that publishes under the same name.

On the Vera node, 8 host actors held 128 sandboxes between them, and 128 creates arrived at once from 64 caller threads, each routed either to a random host or to a host the Ray store listed with a ready sandbox:

| Pools | Routing | Creates that took a booted sandbox | Create p99 | Burst |
| --- | --- | --- | --- | --- |
| 8 × 16 | random host | 119 / 128 | 503 ms | 0.52 s |
| 8 × 16 | store | 119 / 128 | 464 ms | 0.52 s |
| 64, 32, 16, 8, 4, 2, 1, 1 | random host | 72 / 128 | 2,988 ms | 3.01 s |
| 64, 32, 16, 8, 4, 2, 1, 1 | store | 123 / 128 | 176 ms | 0.23 s |
| 64, 32, 16, 8, 4, 2, 1, 1 | store, skipping entries already sent | 127 / 128 | 7 ms | 0.19 s |

A lookup took 0.5 ms with every slot listed, and the store listed every slot ready again 3.5 to 4.5 s after all 128 sandboxes were deleted. A SQLite table of permits behaved the same (127 of 128 with the same routing). Its lookups took 0.22 ms alone, but under 64 concurrent callers, one connection per thread queued them to about 230 ms, and one shared connection per process brought that to about 27 ms, so a database store should share its connections per process.

**Exact claims.** A host hands out any ready sandbox from its queue, so without more, a caller can't tell which listed sandbox a create took. Skipping listed sandboxes it already sent a create for under-counts, because the host may claim a different one and leave a "sent" one ready; counting creates in flight per host over-counts once they return, because a listing lags a claim by up to one publish. In the Vera runs either policy left 1 to 10 of 128 creates cold. So a create can name the record's sandbox, `runtime.create(..., warm_sandbox=entry.sandbox_id)`, and the host claims that one if it's still ready, and otherwise any ready one (ray-project/ray#67728). A caller that sends one create per record then never sends a host more creates than it has sandboxes ready, and a store's rows are permits a caller redeems exactly.

### Multi-sandbox actors

One process can host many sandboxes: the user guide's pool-of-sandboxes actor, a runtime with a warm pool, or the gRPC facade's hosts. Two costs grow with the number of sandboxes per process.

**Open files.** Each sandbox kept two files open in its owner process: its bundle's locked owner file and runsc's stderr log. Every runsc command the process runs forks it, and a fork copies the whole descriptor table: from 64 threads, a process holding 4 descriptors spawned 7,349 commands a second, and one holding 2,004 spawned 329. With 512 sandboxes in one process (1,028 open files), `true` in them ran at 435 a second from 64 threads. Since ray-project/ray#67731, each owning process holds one lock file in `/tmp/ray/sandbox/.owners` for as long as it lives, and each bundle's owner file names it; the stderr log is closed once the sandbox runs, since the launch process writes it through its own descriptors. The same process held 5 files, and ran 753, 927 and 950 commands a second from 32, 64 and 128 threads (499, 435 and 461 before). A bundle's owner is alive while its lock is held and gone once it is free; if the lock file is gone, the PID in its name decides; and an owner that can't be checked counts as alive, because reclaiming a bundle kills its sandbox. An owner that can't delete a bundle (its cgroup won't go) clears the lock from the owner file, so the reclaim pass retries it.

**Actors.** One actor per sandbox, the facade's `SandboxHost` model, costs a Ray worker per sandbox. With 256 sandboxes on the Vera node:

| Layout | All running | Actor memory per sandbox | Threads in the actors | Commands / s |
| --- | --- | --- | --- | --- |
| 256 actors × 1 sandbox | 21.2 s | 154 MiB | 29,696 | 1,019 |
| 32 actors × 8 | 16.0 s | 19.5 MiB | 3,712 | 997 |
| 1 actor × 256 | 11.7 s | 0.64 MiB | 116 | 954 |

Command throughput was the same in all three: the sandboxes set it, not the actors (they were unpinned; see [Sizing](#sizing)). The facade-side form is one node host actor per node that runs all of that node's API sandboxes, with CPU slabs, a node warm pool and a host channel. It is built and benchmarked (on a four-node CoreWeave test cluster, an actor-per-sandbox facade had 117 sandboxes ready a second, node hosts 374, and node hosts with a warm pool 1,069) but not yet proposed as a PR.

### Ownership, reclaim, and runsc's root directory

A sandbox stops with the worker that owns it, and the next create on the node reclaims what an owner that died left behind (ray-project/ray#67022; see [Lifecycle Management](#4-lifecycle-management--garbage-collection)). Since ray-project/ray#67731, liveness comes from per-process owner locks rather than an open file per sandbox.

runsc keeps every container's state and lock file in one root directory (`/tmp/runsc`) and finds a container by globbing it. runsc removes a container's lock file when it destroys the container, but a runsc command that loads the container during that teardown, such as Ray's `runsc delete -force` racing the exit of `runsc run`, creates the file again, and nothing removed it. With release-20261026.0, Ray's deletes left one for 28 of 192 sandboxes, and on a node with 1,483 leftover files, `runsc exec` from 64 threads ran 1,437 times a second against 2,019 with the files gone.

- ray-project/ray#67729 removes a deleted sandbox's lock file once nothing runs runsc for it, and the reclaim pass removes Ray sandboxes' lock files with no state that are older than 10 minutes.
- google/gvisor#16903 stops runsc from making them: a command on a container that must exist no longer creates its lock file, and a lock taken on a file that was removed is retaken on the current one. On gVisor master, Ray's deletes left 2 per 1,920 sandboxes; with the change, none.
- google/gvisor#16902 finds a root container by its full ID without listing the directory: `runsc state` takes 8.4 ms with 50,000 files in the root (30.9 ms before), and commands in 512 sandboxes from 8 processes ran 5.6% faster.

### Start-up path and bursts

- **Bursts are limited by pivot_root(2).** pivot_root(2) walks every thread on the host (`chroot_fs_refs`) under the global `namespace_sem`, and runsc pivots twice per sandbox, in the sentry and in the gofer. On the Vera node one pivot took 1.2 ms with 19,000 threads on the host and 7 ms with 119,000, and each unpinned sandbox adds about 200 threads, so bursts degrade quadratically: 2,000 sandboxes started at once took 383 s, and 18 s without the walk. google/gvisor#16901 switches roots with `MOVE_MOUNT_BENEATH` on Linux 7.1 and later, and falls back to pivot_root(2) elsewhere.
- **Readiness from runsc's pid file** (ray-project/ray#67074): a create returns within about 10 ms of the container existing, instead of after the next `runsc state` poll; sequential creates went from 114 ms to 41 ms.
- **`GOMAXPROCS=2` for runsc CLI calls** (ray-project/ray#67075): each runsc invocation is sized for 2 CPUs, not every CPU on the host (31% more commands a second on 64 cores).
- **Less work per create** (ray-project/ray#67076): a cached `runsc spec`, and image cache hits under a shared lock (3.3 ms to 0.09 ms).
- **No cgroup join per exec** (ray-project/ray#67690): a command in a sandbox with limits started in 49 ms, now 3 ms, because commands no longer queue on a global kernel lock.
- **Overlapping image builds run in child interpreters** (ray-project/ray#67746). Turning an image's layers into its EROFS root goes through `tarfile` twice, both holding the interpreter lock, so a process that pulls several images at once built them one at a time: the 49 Terminal-Bench images (70 GiB) from an empty cache took 265 s with the node 0.7% busy, against about 5 s for one image alone. While one build runs in a process, the next ones now run in child interpreters, at most one per two CPUs, and the 49 pulls took 21 to 22 s.
- On RDMA hosts in the default shared mode, every new network namespace gets a copy of each RDMA device under a mutex, which caps creates at about 120 to 150 a second on the Vera node once the pivot_root walk is gone. Nodes where no pod needs shared RDMA devices should boot with `ib_core.netns_mode=0`.

### Sizing

An unpinned sandbox (`cpu=0`) sees every CPU on the host, and gVisor sizes its Go runtime and thread pools for all of them: an idle one holds about 201 kernel tasks, against 24 at `cpu=2`. On the shared 352-CPU host, sandboxes that each saw every CPU lost about half their throughput: Terminal-Bench oracle runs at `cpu=2` reached 95% of Docker's runs per minute with 147 lanes (8,207 a minute), and 11,464 a minute with 294 lanes, while unpinned sandboxes reached 48%. With 512 unpinned sandboxes, commands in them topped out near 1,000 a second from any number of processes, with the node 74% busy and 172,000 threads on it. Dense nodes should size sandboxes with `cpu=`.

A sized sandbox runs on the least-used hardware threads, physical cores first, so once more sandboxes have limits than the node has cores, they share cores: a `cpu=2` probe among 160 busy `cpu=2` neighbors ran 1.86× slower. `RAY_SANDBOX_CPU_PLACEMENT=core` (ray-project/ray#67747) gives each sandbox whole physical cores instead, as Kubernetes' CPU manager does with `full-pcpus-only`; the probe then ran 1.04× slower, and Terminal-Bench throughput with 294 `cpu=2` sandboxes stayed the same (11,488 and 11,367 runs a minute).

### Node settings, arm64, and tests

- `RAY_SANDBOX_DEFAULT_DNS` (ray-project/ray#67691) lets a node replace `network="public"` sandboxes' default resolvers where public DNS is blocked.
- `RAY_SANDBOX_RUNSC_FLAGS` (ray-project/ray#67692) lets a node pass extra global flags to runsc, such as `--platform=kvm`. KVM made system calls about 5× cheaper for a single sandbox on the Vera node. gVisor's arm64 KVM platform ran trapped `WFE` instructions, which OpenBLAS uses to spin-wait, and reads of ID registers as system calls, which crashed PyTorch; with google/gvisor#16909, all 49 Terminal-Bench tasks pass on KVM at 0.98× Docker's time with `cpu=1`. About 0.7% of KVM sandboxes that read random data also died, sized or not, which is why unsized ones died under load: gVisor left host pointer authentication on for about one thread in 128, so the host vDSO's `getrandom` failed its return-address check when a guest exit interrupted it. Upstream gVisor fixed that on November 3 (google/gvisor@dcae68e54, not yet in a release); on the Vera node, deaths went from 203 in 29,716 sandboxes to none in 62,284. Churn also queued dead VMs: gVisor releases each sandbox's VM from a kernel worker, and the kernel destroys those one at a time, here 50-70 a second, so churning faster than that left 99,097 VMs and about 1.4 TB queued until the node ran out of memory; google/gvisor#17029 releases them in parallel. KVM still needs those fixes and google/gvisor#16909, and sized sandboxes lose throughput once their vCPUs, two each, outnumber the host's CPUs, so it stays opt-in.
- `RAY_SANDBOX_PIDS_MAX` (ray-project/ray#67748) caps the host tasks each sandbox may run. Under gVisor's systrap platform a guest process costs about two host tasks, so without a cap one sandbox could use up the node's process IDs: 2,000 processes in one sandbox took 4,035 host tasks. With a cap of 1,024 they stopped at 1,008, and the sandbox and its neighbor kept running.
- On arm64, gVisor started every application with speculative store bypass disabled; google/gvisor#16814 sets SSBS, which makes CPU-bound code 2.1× faster. On hosts with 16 or 64 KiB pages, gVisor's vDSO looked for its clock parameters in the wrong place, so every clock read became a system call; google/gvisor#16815 fixes that (3.4 µs to 0.24 µs per read). gVisor also showed applications no HWCAP2 feature, so numeric libraries skipped BF16 and I8MM, and PyTorch's bfloat16 matrix multiplies ran 220 to 490× slower than with them; google/gvisor#16910 shows applications the HWCAP2 features that keep no state of their own.
- A sandbox's PID 1 was `sleep infinity`, which never reaps, so every process whose parent exited stayed a zombie: 1,000 commands that each left a background process behind left 1,000, and a sandbox has 65,536 process IDs. With ray-project/ray#67749, PID 1 is a shell loop that reaps them, and commands keep their own exit codes.
- ray-project/ray#67694 tests that a sandbox that runs out of memory is killed whole while its neighbors, the node and the parent cgroup are untouched.

### Future work from these changes

- The facade's node host actors, and facade warm pools that publish to the store.
- Per-sandbox or sharded runsc roots: `runsc run` boot time grew with the number of sandboxes on a node (p50 47 ms at 100 sandboxes, about 150 to 175 ms at 3,100), most likely from the shared `/tmp/runsc` directory.
- A default vCPU count for unpinned sandboxes, so one doesn't size itself to the whole host. This changes what `cpu=0` means to the guest, so it needs a product decision.

---

## Compatibility, Deprecation, and Migration Plan

- **Backwards Compatibility**: This change is 100% backwards compatible. All APIs are new and experimental. No existing Ray Core APIs or behaviors are modified or deprecated.
- **Migration Path**: Frameworks currently using custom subprocess wrappers or external REST APIs can directly replace their implementation with one of the Ray experimental APIs.

---

## Test Plan and Acceptance Criteria

### Unit Tests
- `ray.experimental.sandbox` tests
- `ray.experimental.Sandbox` tests
- `ray.experimental.SandboxRuntime` tests
- `GVisorSandboxRuntime` OCI spec generation and CLI command string verification (`runsc create`, `start`, `exec`, `kill`, `delete`).
- Configuration validation (invalid resource specs, timeouts, image names).

### Integration & E2E Tests
- **gVisor Runtime Testing**: E2E tests on nodes with `runsc` installed:
  - **Startup Latency**: Benchmark sandbox instantiation to verify sub-100ms startup times.
  - **Bin Packing Density**: Verify execution of 100+ concurrent gVisor sandboxes on a single worker node.
  - **Command Execution & Files**: Test `runsc exec`, file upload/download, stdout/stderr capture, and timeout enforcement.
- **Security Validation**: Verify that non-root, read-only rootfs, and dropped capability settings prevent privilege escalation inside gVisor sandboxes.
- **Orphan Sandbox Cleanup**: Test process crashes (killing driver script or worker process) and verify that orphan `runsc` sandbox containers are cleaned up cleanly.
- **Real-gVisor suites**: The tests that need `runsc` (`TEST_SANDBOX=1`) also run in a privileged container with a pinned gVisor release, as well as on nodes with 64 KiB pages.

### Benchmarks
Each design change above is measured, not just tested, on a 352-CPU node and on Kubernetes clusters:
- **Creates**: sequential latency, and bursts of 500, 1,000 and 2,000 sandboxes at once (makespan, errors, where the time goes).
- **Density**: idle sandboxes per node and memory per sandbox (2,185 on the Vera node at 97 MiB each, limited by the container's task count; on four 60-CPU pods, 15,990 at about 26 MiB each).
- **Commands**: `true` per second against sandboxes per process and against concurrency.
- **Warm pools**: claim latency for a burst as large as the pool, refill time, and that a pool never exceeds its size; store routing across host actors against random routing.
- **Multi-sandbox actors**: time to running, actor memory per sandbox, and command throughput for one actor per sandbox through one actor for all.
- **Containment**: CPU quotas under load, out-of-memory kills, disk caps and task caps with real cgroups, and a latency probe among busy neighbors.
- **Images**: a different image per sandbox, cold and warm, and a cache capped below its contents while some images are in use.
- **Workloads**: Terminal-Bench 2.1 oracle runs per minute against lanes, sandbox sizing, CPU placement and gVisor platform; stock Modal SDK and Harbor clients through the gRPC facade; SkyRL rollouts on H100s with every sandbox on the Vera node (256 trials at once, no sandbox failures, sandbox setup p50 1.6 s).

### Acceptance Criteria
1. Complete `ray.experimental.sandbox` Python package exported under `ray.experimental.sandbox`.
2. Complete implementation for `ray.experimental.Sandbox` actor.
3. Complete implementation for `ray.experimental.SandboxRuntime` with a fully functional `gvisor` sandbox implementation supporting sandbox lifecycle, command execution, and file transfers.
4. Comprehensive documentation and example script showing RL code evaluation using all experimental APIs.

### GA Graduation

On GA gradution, all experimental APIs in `ray.experimental.sandbox` will be migrated to a top-level `ray.sandbox` package.
All experimental APIs are subject to breaking changes or removal prior to GA graduation.

---

## (Optional) Follow-on Work

- **Additional Runtimes**: Support for other sandboxing runtimes such as [Agent Substrate](https://github.com/agent-substrate/substrate), Firecracker microVMs, and third-party APIs (modal, etc.).
- **Checkpoint / Restore**: Support for checkpoint / restoring sandboxes, leveraging `runsc checkpoint` and `runsc restore` for state snapshotting and sub-10ms execution start.
- **Modal Compatible API:** Support a Modal compatible API in `ray.experimental.sandbox.modal`.
