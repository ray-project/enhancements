## Summary
### General Motivation

Currently, we don't control and limit the resource usage of worker processes, except running the worker in a container (see the `container` part in the [doc](https://docs.ray.io/en/latest/ray-core/handling-dependencies.html#api-reference)). In most of the scenarios, the container is unnecessary, but resource control is necessary for isolation.

[Control groups](https://man7.org/linux/man-pages/man7/cgroups.7.html), usually referred to as cgroups, are a Linux kernel feature which allow processes to be organized into hierarchical groups whose usage of various types of resources can then be limited and monitored.

So, the goal of this proposal is to achieve resource control for worker processes by cgroup in Linux.

### Should this change be within `ray` or outside?

These changes would be within Ray Core.

## Stewardship
### Required Reviewers
@ericl, @edoakes, @simon-mo, @chenk008 @raulchen 

### Shepherd of the Proposal (should be a senior committer)
@ericl

## Design and Architecture

### Cluster level API
We should add some new system configs for the resource control.
- worker_resource_control_method: Set to "cgroup" by default.
- cgroup_manager: Set to "cgroupfs" by default. We should also support `systemd`.
- cgroup_mount_path: Set to `/sys/fs/cgroup/` by default.
- cgroup_unified_hierarchy: Whether to use cgroup v2 or not. Set to `False` by default.
- cgroup_use_cpuset: Whether to use cpuset. Set to `False` by default.

### User level API
Users could turn on/off resource control by setting the relevant fields of runtime env (in job level or task/actor level).
#### Simple API using a flag
```python
    runtime_env = {
        "enable_resource_control": True,
    }
```

```python
    from ray.runtime_env import RuntimeEnv
    runtime_env = ray.runtime_env.RuntimeEnv(
        enable_resource_control=True
    )
```

#### Entire APIs
```python
    runtime_env = {
        "enable_resource_control": True,
        "resource_control_config": {
            "cpu_enabled": True,
            "memory_enabled": True,
            "cpu_strict_usage": False,
            "memory_strict_usage": True,
        }
    }
```

```python
    from ray.runtime_env import RuntimeEnv, ResourceControlConfig
    runtime_env = ray.runtime_env.RuntimeEnv(
        resource_control_config=ResourceControlConfig(
            cpu_enabled=True, memory_enabled=True, cpu_strict_usage=False, memory_strict_usage=True)
    )
```

### How to manage cgroups
#### cgroup versions
The cgroup has two versions: cgroup v1 and cgroup v2. [Cgroup v2](https://www.kernel.org/doc/Documentation/cgroup-v2.txt) was made official with the release of Linux 4.5. But only a few systems are known to use croup v2 by default: Fedora (since 31), Arch Linux (since April 2021), openSUSE Tumbleweed (since c. 2021), Debian GNU/Linux (since 11) and Ubuntu (since 21.10). It means that, although a lot of persons are using the Linux with high versions, they can't use cgroup v2 if they don't make a special configration and reboot.

Check if cgroup v2 has been enabled in your linux system:
```
mount | grep '^cgroup' | awk '{print $1}' | uniq
```

And you can try to enable cgroup v2:
```
grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
reboot
```

[Cgroup v2](https://www.kernel.org/doc/Documentation/cgroup-v2.txt) seems more reasonable and easy to use, but we should also support cgroup v1 because v1 is widely used and has been hard coded to the [OCI](https://opencontainers.org/) standards. You can see this [blog](https://www.redhat.com/sysadmin/fedora-31-control-group-v2) for more.

#### cgroupfs
The traditional way to manage cgroups is writing the file system directly. It is usually referred to as cgroupfs, like:
```
mkdir /sys/fs/cgroup/{worker_id} # Create cgroup
echo "200000 1000000" > /sys/fs/cgroup/foo/cpu.max # Set cpu quota
echo {pid} > /sys/fs/cgroup/foo/cgroup.procs # Bind process.
```
NOTE: This is an example based on cgroup v2. The commmand lines in cgroup v1 is defferent and incompatible.

We also could use the [libcgroup](https://github.com/libcgroup/libcgroup/blob/main/README) to simplify the implementation. This library support both cgroup v1 and cgroup v2. 

#### systemd
Systemd, the init daemon of most of linux hosts, is also a cgroup driver which can use cgroup to manage processes. It provide a wrapped way to bind process and cgroup, like:
```
systemctl set-property {worker_id}.service CPUQuota=20% # Bind process and cgroup automatically.
systemctl start {worker_id}.service
```

NOTE: The entire config options is [here](https://man7.org/linux/man-pages/man5/systemd.resource-control.5.html). And we can also use `StartTransientUnit` to create cgroup with worker process simply. This is a [dbus](https://www.freedesktop.org/wiki/Software/systemd/dbus/) API and there is a [dbus-python](https://dbus.freedesktop.org/doc/dbus-python/) module we can used.

#### Why we should support both cgroupfs and systemd
The cgroupfs and the systemd are two mainstream ways to manage cgroups. In [container technology](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/), this two ways are also supported. We should also support both of them.

The reason of using cgroupfs:
- Cgroupfs is a traditional and common way to manage cgroups. If your login user has the privilege of cgroup file system, you can create the cgroups.
- In some environments, you couldn't use systemd, e.g. in a non-systemd based system or in a container which couldn't access the systemd service.

The reason of using systemd:
- Systemd is highly recommended by [runc](https://github.com/opencontainers/runc/blob/main/docs/cgroup-v2.md.) for cgroup v2.
- If we use systemfs in a systemd based system, there will be more than one component which manage the cgroup tree.
- In the systemd [docs](https://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/), we can know that  create/delete cgroups will be unavailable to userspace applications, unless done via systemd's APIs.
- The systemd has a good abstract API of cgroups. 

### Support using cgroups in Ray

1. Cgroup manager in the Agent

The Cgroup Manager is used to create or delete cgroups, and bind worker processes to cgroups. We plan to integrate the Cgroup Manager in the Agent. 

We should abstract the Cgroup Manager because we have more than one way to manager cgroups in linux: cgroupfs and systemd.

2. Generate the command line of cgroup.

Agent should generete the command line from cgroup manager. The command line will be append to the `command_prefix` field of runtime env context. A command line sample is like:
    `mkdir /sys/fs/cgroup/{worker_id} && echo "200000 1000000" > /sys/fs/cgroup/foo/cpu.max && echo {pid} > /sys/fs/cgroup/foo/cgroup.procs`

3. Start worker process with the command line.

Currently, we use `setup_worker.py` to enforce the runtime environments. `setup_worker.py` will merge the `command_prefix` of runtime env context and real worker command. In the same way, the command about cgroup will be run with worker process.

4. Delete cgroup.

When worker processes die, we should delete the cgroup which is created for the processes. This work is only needed for cgroupfs because systemd could handle it.

So, we should add a new RPC named `CleanForDeadWorker` in `RuntimeEnvService`. The Raylet should send this PRC to the Agent and the Agent will delete the cgroup.

```
message CleanForDeadWorkerRequest {
  string worker_id = 1;
}

message CleanForDeadWorkerReply {
  AgentRpcStatus status = 1;
  string error_message = 2;
}

service RuntimeEnvService {
...
  rpc CleanForDeadWorker(CleanForDeadWorkerRequest)
      returns (CleanForDeadWorkerReply);
}
```

## Compatibility, Deprecation, and Migration Plan

This proposal will not change any existing APIs and any default behaviors of Ray Core.

## Test Plan and Acceptance Criteria

We plan to benchmark the resouces of worker processes:
- CPU soft control: The worker could use idle CPU times which exceeds the CPU quota.
- CPU hard control: Set the config `cpu_strict_usage=True`. The worker couldn't exceed the CPU quota.
- Memory soft control: The worker could use idle memory which exceeds the memory quota.
- Memory hard control: Set the config `memory_strict_usage=True`. The worker couldn't exceed the memory quota.

Acceptance criteria:
- A set of reasonable APIs.
- A set of reasonable benchmark results.

## (Optional) Follow-on Work

In the first version, we can only support one cgroup manager based on cgroupfs. We can support systemd based cgroup manager in future.
And we can achieve control for more resources, like `blkio`, `devices`, and `net_cls`.

## Appendix
### The Work steps of resource control based on cgroup.

When we run `ray.init` with a `runtime_env` and `eager_install` is enabled, the main steps are:
  - (**Step 1**) The Raylet(Worker Pool) receives the publishing message of job started.
  - (**Step 2**) The Raylet sends the RPC `GetOrCreateRuntimeEnv` to the Agent. 
  - (**Step 3**) The Agent setups the `runtime_env`. For `resource_control`, the agent generates `command_prefix` in `runtime_env_context` to talk how to enable the resource control.

When we create a `Task` or `Actor` with a `runtime_env`(or a inherited `runtime_env`), the main steps are:
  - (**Step 4**) The worker submits the task.
  - (**Step 5**) The `task_spec` is received by the Raylet after scheduling.
  - (**Step 6**) The Raylet sends the RPC `GetOrCreateRuntimeEnv` to the Agent.
  - (**Step 7**) The agent generates `command_prefix` in `runtime_env_context` and replies the RPC.
  - (**Step 8**) The Raylet starts the new worker process with the `runtime_env_context`.
  - (**Step 9**) The `setup_worker.py` setups the `resource_control` by `command_prefix` for the new worker process.

### More references
- [Yarn NodeManagerCgroups](https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/NodeManagerCgroups.html)
