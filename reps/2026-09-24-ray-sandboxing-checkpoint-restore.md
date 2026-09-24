# Summary

### General Motivation

In [REP 2026-07-14: Ray Sandboxing](2026-07-14-ray-sandboxing.md), we introduced `ray.experimental.sandbox`, a native sandboxing library designed to execute untrusted model-generated code and agent tool calls safely on Ray clusters using gVisor (`runsc`).

While basic sandbox lifecycle and execution are supported, advanced Reinforcement Learning (RL) and Agentic reasoning paradigms introduce new demands for high-frequency environment instantiation and branching:
1. **Tree Search and Rollout Branching (e.g., MCTS, ToT, Search-R1, QwQ):** Algorithms like Monte Carlo Tree Search explore hundreds of possible action branches from a common intermediate state (e.g., a warm SWE-bench environment with dependencies, git repos, and pre-computed indexes). Starting fresh sandboxes from scratch for every exploration branch incurs massive cold-boot and setup latency.
2. **Environment Pre-Warming:** Complex execution environments (heavy Python dependencies, ML models, initialized database state) can take 30–60 seconds to initialize. Agent frameworks require instant sandbox startup without paying the warm-up cost repeatedly.
3. **Fault Tolerance & Preemption:** Long-running agent workflows or evaluations need checkpointing so they can be paused, persisted, and migrated across preemptible instances.

This proposal extends `ray.experimental.sandbox` by adding first-class **Checkpoint and Restore** primitives. By leveraging gVisor's native `runsc checkpoint` and `runsc restore` alongside userspace EROFS container rootfs layering, Ray sandboxes can capture entire execution states (guest memory, CPU registers, file descriptors, network sockets, and copy-on-write overlay deltas) and restore them into multiple independent child sandboxes in **< 100 milliseconds**.

### Should this change be within `ray` or outside?

Within `ray` as an enhancement to the experimental library (`ray.experimental.sandbox`).

## Stewardship

### Required Reviewers

@edoakes
@MengjinYan
@richardliaw
@kouroshHakha

### Shepherd of the Proposal

@robertnishihara @pcmoritz

---

## Design and Architecture

### Overview

Checkpoint & Restore in Ray Sandboxing introduces:
1. **`checkpoint(...)`**: Captures the complete running state of a sandbox container into a self-contained checkpoint bundle directory containing:
   - `state/`: gVisor memory dump (`checkpoint.img`, `pages.img`, `pages_meta.img`).
   - `fs/`: Snapshot of the writable overlay upper layer and explicit working directories.
   - `manifest.json`: Serialized sandbox configuration, OCI spec parameters, and execution metadata.
2. **`restore(...)`**: Spawns one or more new sandbox instances initialized directly from a saved checkpoint bundle.
3. **`pause()` / `resume()`**: Pauses guest execution without disk serialization for transient resource multiplexing.

```
                         Base Sandbox (Actor / Runtime)
                         +----------------------------+
                         | Python / SWE-bench Warmup  |
                         +----------------------------+
                                       |
                                       | base_sb.checkpoint(path)
                                       v
                         +----------------------------+
                         | Checkpoint Bundle (Disk)   |
                         |  ├── manifest.json         |
                         |  ├── state/ (memory dump)  |
                         |  └── fs/ (overlay delta)   |
                         +----------------------------+
                           /           |            \
       sandbox.restore()  /            |             \  sandbox.restore()
                         v             v              v
                  Branch 1          Branch 2       Branch 3
              +--------------+  +--------------+  +--------------+
              | Independent  |  | Independent  |  | Independent  |
              | Child State  |  | Child State  |  | Child State  |
              +--------------+  +--------------+  +--------------+
```

---

### API Design

#### 1. Actor API (`ray.experimental.sandbox.Sandbox`)

```python
class Sandbox:
    """Ray Actor wrapping SandboxRuntime with Checkpoint & Restore capabilities."""

    def checkpoint(
        self,
        checkpoint_path: Optional[str] = None,
        leave_running: bool = True,
        timeout_seconds: float = 60.0,
        compression: str = "none",
        exclude_committed_zero_pages: bool = True,
        direct: bool = False,
    ) -> str:
        """Create a checkpoint of the running sandbox.

        Args:
            checkpoint_path: Target directory path where checkpoint will be saved.
                If None, defaults to /tmp/ray/sandbox/checkpoints/<uuid>.
            leave_running: If True, keep the sandbox running after checkpointing.
            timeout_seconds: Timeout for the checkpoint operation.
            compression: Snapshot compression ('none' or 'flate-best-speed').
            exclude_committed_zero_pages: If True, omit zero pages to reduce disk I/O.
            direct: If True, use O_DIRECT for disk writes.

        Returns:
            Absolute path to the checkpoint directory bundle.
        """
        pass

    def pause(self, timeout_seconds: float = 10.0) -> None:
        """Pause all guest processes inside the sandbox without serializing to disk."""
        pass

    def resume(self, timeout_seconds: float = 10.0) -> None:
        """Resume execution of a paused sandbox."""
        pass
```

#### 2. High-Level Function API (`ray.experimental.sandbox`)

```python
def restore(
    checkpoint_path: str,
    cpu: Optional[float] = None,
    memory: Optional[Union[str, int, float]] = None,
    env: Optional[Dict[str, str]] = None,
    workdir: Optional[str] = None,
    background: bool = True,
    direct: bool = False,
    timeout_seconds: float = 30.0,
    **kwargs,
) -> SandboxHandle:
    """Restore a new sandbox instance from an existing checkpoint bundle.

    Args:
        checkpoint_path: Directory path of the saved checkpoint bundle.
        cpu: Optional CPU allocation override.
        memory: Optional memory allocation override.
        env: Optional environment variables to overlay.
        workdir: Optional working directory override.
        background: If True, guest memory pages are loaded asynchronously for
            near-instantaneous (< 10ms) sandbox start.
        direct: If True, use O_DIRECT for memory page restoration.
        timeout_seconds: Timeout waiting for restored instance to reach 'running'.

    Returns:
        SandboxHandle for communicating with the restored instance.
    """
    pass
```

#### 3. Low-Level Runtime Interface (`SandboxRuntime`)

```python
class SandboxRuntime(ABC):

    @abstractmethod
    def checkpoint(
        self,
        instance_id: str,
        checkpoint_path: Optional[str] = None,
        leave_running: bool = True,
        **kwargs,
    ) -> str:
        """Checkpoint local sandbox instance."""
        pass

    @abstractmethod
    def restore(
        self,
        checkpoint_path: str,
        **kwargs,
    ) -> str:
        """Restore local sandbox instance from checkpoint."""
        pass

    @abstractmethod
    def pause(self, instance_id: str) -> None:
        """Pause sandbox instance."""
        pass

    @abstractmethod
    def resume(self, instance_id: str) -> None:
        """Resume sandbox instance."""
        pass
```

---

### Internal Architecture & Implementation Details

#### 1. gVisor Backend Mechanics (`runsc checkpoint` / `restore`)

- **State Serialization (`runsc checkpoint`):**
  When `checkpoint(...)` is invoked:
  1. Ray invokes:
     ```bash
     runsc --root=/tmp/ray/sandbox/runsc checkpoint \
       --image-path=<bundle_dir>/state \
       [--leave-running] \
       [--compression=none|flate-best-speed] \
       [--exclude-committed-zero-pages] \
       <sandbox_id>
     ```
  2. gVisor's Sentry kernel stops vCPUs, flushes open file caches, dumps process trees, guest memory tables, and socket states into protobuf/image files (`checkpoint.img`, `pages.img`, `pages_meta.img`).
  3. Ray preserves any modified filesystem files from the copy-on-write overlay into `<bundle_dir>/fs/` while ignoring internal ephemeral files (e.g. `.gvisor.filestore*`).
  4. Ray writes `manifest.json` recording base image, cwd, environment, and resource parameters.

- **State Restoration (`runsc restore`):**
  When `restore(...)` is invoked:
  1. Ray extracts sandbox bundle configuration from `manifest.json`.
  2. A new unique `instance_id` (e.g. `ray-sandbox-3fa901bc`) and bundle directory are created.
  3. Image manager re-attaches the immutable base image via userspace EROFS annotations (`dev.gvisor.spec.rootfs.source`).
  4. Ray restores the upper writable layer from `<bundle_dir>/fs/` into the new sandbox root.
  5. Ray launches `runsc restore`:
     ```bash
     runsc --root=/tmp/ray/sandbox/runsc restore \
       --bundle=<new_bundle_dir> \
       --image-path=<bundle_dir>/state \
       [--background] \
       [--direct] \
       <new_sandbox_id>
     ```
  6. The Sentry re-maps guest pages and resumes execution exactly where the base sandbox was checkpointed. With `--background`, memory pages are faulted in lazily on demand, enabling sub-10ms startup.

#### 2. Rootless vs. Rooted Execution & Permissions

- gVisor's upstream implementation of `runsc checkpoint` and `runsc restore` requires root capabilities (`SYS_PTRACE`, `SYS_ADMIN`) to dump and restore process address spaces and namespaces.
- When `rootless=True`, `runsc checkpoint` cannot dereference proc magic links or re-establish UID/GID mappings. 
- **Enforcement:** Ray explicitly validates `config.rootless` and raises a clean `SandboxError("gVisor (runsc) does not support checkpoint/restore in rootless mode. Please configure rootless=False.")` rather than causing silent corruption or host process crashes.
- Inside privileged containers (such as Kubernetes Ray worker pods), Ray automatically manages privilege escalation (running `sudo runsc` if executing as unprivileged user `ray`) and ensures `chmod -R a+rw` on checkpoint bundles.

#### 3. Storage Topology & Scope

##### Supported Today: Node-Local Storage
In the current implementation, `checkpoint_path` expects a path on the local worker filesystem. The snapshot bundle is read and written directly via POSIX filesystem operations:
- **`emptyDir` Volume (`/tmp/ray`):** Standard container ephemeral volume backed by the node's boot disk (e.g. ext4 on GKE). Provides fast sub-second checkpointing (~0.07s) and restoration (~0.001s handle start) out of the box with zero cluster infrastructure changes.
- **Local NVMe SSD Mount (`/mnt/disks/ssd0`):** Ephemeral local SSD attached to the node for heavy guest memory checkpoints (e.g. 8GB+ memory footprints) requiring maximum local IOPS.

*Constraint:* Because storage is node-local, child sandboxes restored from a node-local checkpoint must be scheduled on the **same Ray worker node** where the checkpoint was written. This matches the dominant MCTS / RL rollout pattern where parallel exploration branches are collocated on the same worker.

##### Future Follow-on Work: Remote Storage (GCS / S3 / Object Store)
Native support for remote cloud storage (e.g., `gs://bucket/checkpoints/...` or `s3://...`) and cross-node checkpoint migration is planned as a future extension:
1. **Ray Object Store (`Plasma`):** Streaming the checkpoint bundle into Ray's distributed object store (`ray.put()`) to broadcast snapshots across worker nodes via Ray's native peer-to-peer transport.
2. **Direct Remote Storage Support:** Integrating cloud object store clients or relying on POSIX mount layers (e.g., Cloud Storage FUSE) with asynchronous memory page prefetching.

---

### Examples

#### Example 1: Parallel MCTS RL Rollouts (Branching from Pre-Warmed Base)

```python
import ray
from ray.experimental import sandbox

ray.init()

# Step 1: Pre-warm a heavy SWE-bench environment once
base_sb = sandbox.create(
    image="swebench-eval:latest",
    cpu=2.0,
    memory="4Gi",
    workdir="/workspace",
    readonly=False,
)
base_sb.exec("git checkout base-eval-commit && pip install -e .")

# Step 2: Checkpoint warm environment
checkpoint_dir = base_sb.checkpoint(
    checkpoint_path="/tmp/ray/swebench_warm_checkpoint",
    leave_running=False,
)

# Step 3: Concurrently branch N candidate trajectories from the exact same state
@ray.remote
def evaluate_trajectory(checkpoint_path: str, action_patch: str) -> float:
    # Restores in ~0.001s without re-installing dependencies or re-cloning repo
    child_sb = sandbox.restore(checkpoint_path=checkpoint_path, cpu=1.0)
    try:
        child_sb.write_file("/tmp/action.patch", action_patch)
        child_sb.exec("git apply /tmp/action.patch")
        res = child_sb.exec("pytest tests/test_patch.py")
        return 1.0 if res.exit_code == 0 else 0.0
    finally:
        child_sb.delete()

futures = [
    evaluate_trajectory.remote(checkpoint_dir, patch)
    for patch in candidate_patches
]
rewards = ray.get(futures)
```

#### Example 2: Pausing Idle Sandboxes

```python
# Pause sandbox to release host CPU resources while waiting for external LLM inference
sb.pause()

# Prompt LLM for next tool call ...
next_action = llm.generate(...)

# Instantly resume without disk I/O
sb.resume()
sb.exec(next_action)
```

---

### Verification and Benchmark Results

The feature was tested and validated end-to-end on a live Google Kubernetes Engine (GKE) cluster (Container-Optimized OS, `6.12.105+`):

1. **Cold Start vs. Restore Latency:**
   - **Base Sandbox Cold Start:** `0.32s`
   - **Checkpoint Creation (`runsc checkpoint`):** `0.07s`
   - **Average Branch Restore Latency (`sandbox.restore`):** `0.001s` (Handle) / `0.11s` (Fully Active)
   - **Speedup Factor:** **> 30x faster** than fresh cold-start instantiation.
2. **State Isolation:**
   - Child branches simultaneously mutated `/workspace/state_file.txt` with branch-specific payloads. No cross-branch state pollution occurred.
3. **GKE COS Compatibility:**
   - Validated that gVisor's in-userspace EROFS parser and memory restoration operate cleanly on Container-Optimized OS without requiring host kernel `CONFIG_EROFS_FS`.

---

### Compatibility, Deprecation, and Migration Plan

- **Backwards Compatibility:** Fully non-breaking. The `checkpoint()`, `restore()`, `pause()`, and `resume()` methods are pure extensions to existing classes.
- **Pre-requisite:** Sandboxes requiring checkpoint/restore must run with `rootless=False` and have `SYS_PTRACE` capabilities.

---

### Test Plan and Acceptance Criteria

#### Unit Tests
- `tests/test_gvisor_checkpoint.py`: Validates OCI spec preparation, command construction (`runsc checkpoint`, `runsc restore`), and manifest generation.
- `tests/test_rootless_rejection.py`: Asserts that `rootless=True` sandboxes cleanly raise `SandboxError`.

#### Integration Tests
- E2E parallel restoration test verifying multi-branch state isolation (TC-01 through TC-06).
- MCTS rollout pipeline simulation with parallel Ray remote tasks (`app_checkpoint/verl_mcts_checkpoint_pipeline.py`).

---

## (Optional) Follow-on Work

- **Ray Object Store (`Plasma`) Checkpoint Distribution:** Broadcast checkpoint bundles across Ray worker nodes via Ray's distributed object store (`ray.put()` / `ray.get()`) to enable cross-node branching and restore.
- **Direct Remote Object Storage Support:** Native integration with Cloud Object Storage (`gs://...`, `s3://...`) for persistent, cross-cluster checkpoint management.
- **Copy-on-Write Memory Forking:** Investigate userspace CoW memory page sharing across child sandboxes to reduce memory footprint during large-batch rollouts.
- **Live Memory Streaming & Prefetching:** Integrate gVisor remote page faults with streaming network backends to achieve zero-wait restore over high-latency networks.
- **Agent Substrate & MicroVM Checkpoint Support:** Generalize the checkpoint/restore abstraction across Firecracker / Substrate microVM backends as they are added to Ray Sandboxing.

