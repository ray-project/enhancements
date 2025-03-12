## Summary
### General Motivation

Currently, we don't control and limit the resource usage of worker processes, except running the worker in a container (see the `container` part in the [doc](https://docs.ray.io/en/latest/ray-core/handling-dependencies.html#api-reference)). In most of the scenarios, the container is unnecessary, but resource control is necessary for isolation.

[Control groups](https://man7.org/linux/man-pages/man7/cgroups.7.html), usually referred to as cgroups, are a Linux kernel feature which allow processes to be organized into hierarchical groups whose usage of various types of resources can then be limited and monitored.

So, the goal of this proposal is to achieve resource control for worker processes by cgroup in Linux.

### Should this change be within `ray` or outside?

The resource control ability will be implemented in a runtime environments plugin. So the changes should no longer be within Ray core, but scoped to ray.experimental.cgroup_env_plugin (moved to util eventually).

## Stewardship
### Required Reviewers
@ericl, @edoakes, @simon-mo, @chenk008 @raulchen 

### Shepherd of the Proposal (should be a senior committer)
@edoakes

## Design and Architecture

### Cluster level API
We should add some new environment variables for configuration in the `cgroup_env_plugin`.
- RAY_CGROUP_MANAGER: Set to "cgroupfs" by default. We should also support `systemd`.
- RAY_CGROUP_MOUNT_PATH: Set to `/sys/fs/cgroup/` by default.
- RAY_CGROUP_UNIFIED_HIERARCHY: Whether to use cgroup v2 or not. Set to `False` by default.
- RAY_CGROUP_USE_CPUSET: Whether to use cpuset. Set to `False` by default.

### User level API
Users could turn on/off resource control by setting the relevant fields of runtime env (in job level or task/actor level).

```python
    runtime_env = {
        "plugins": {
            "resource_control": {
                "enable": True,
                "cpu_enabled": True,
                "memory_enabled": True,
                "cpu_strict_usage": False,
                "memory_strict_usage": True,
            }
        }
    }
```

```python
    from ray.runtime_env import RuntimeEnv
    runtime_env = ray.runtime_env.RuntimeEnv(
        plugins={
            "resource_control": {
                "enable": True,
                "cpu_enabled": True,
                "memory_enabled": True,
                "cpu_strict_usage": False,
                "memory_strict_usage": True,
            }
        }
    )
```

The meanings of the config items:
- enable: The switch of the resource control. Set to `False` by default.
- cpu_enabled: Enable the cpu resource control. The cpu quota is set from scheduling, e.g. from the `num_cpus` option of `@ray.remote(num_cpus=1)`. This is a soft limit which means that, it doesn't limit the worker's cpu usage if the cpus are not busy.
- memory_enabled: Enable the memory resource control. Same as `cpu_enabled`, this is also a soft limit.
- cpu_strict_usage: Limit the cpu resource strictly. Workers couldn't exceed the cpu quota even though the cpus are not busy.
- memory_strict_usage: Limit the memory resource strictly which is the same as `cpu_strict_usage`.

### How to manage cgroups
#### cgroup versions
The cgroup has two versions: cgroup v1 and cgroup v2. [Cgroup v2](https://www.kernel.org/doc/Documentation/cgroup-v2.txt) was made official with the release of Linux 4.5. But only a few systems are known to use croup v2 by default(refer to [here](https://rootlesscontaine.rs/getting-started/common/cgroup2/)): Fedora (since 31), Arch Linux (since April 2021), openSUSE Tumbleweed (since c. 2021), Debian GNU/Linux (since 11) and Ubuntu (since 21.10). For other Linux distributions, even if they use newer Linux kernels, users still need to change a configuration (see below) and reboot the system to enable cgroup v2.

Check if cgroup v2 has been enabled in your linux system:
```
mount | grep '^cgroup' | awk '{print $1}' | uniq
```

NOTE: Cgroup v2 is also not supported in the Ray released images, e.g. `rayproject/ray:latest`.

And you can try to enable cgroup v2:
```
grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
reboot
```

Overall, cgroup v2 has better design and is easier to use, but we should also support cgroup v1 because v1 is widely used and has been hard coded to the [OCI](https://opencontainers.org/) standards. See this [blog](https://www.redhat.com/sysadmin/fedora-31-control-group-v2) for more information.

#### cgroupfs
The traditional way to manage cgroups is writing the file system directly. It is usually referred to as cgroupfs, like:
```
mkdir /sys/fs/cgroup/{worker_id} # Create cgroup
echo "200000 1000000" > /sys/fs/cgroup/foo/cpu.max # Set cpu quota
echo {pid} > /sys/fs/cgroup/foo/cgroup.procs # Bind process.
```
NOTE: This is an example based on cgroup v2. The command lines in cgroup v1 is different and incompatible.

We can also use the [libcgroup](https://github.com/libcgroup/libcgroup/blob/main/README) to simplify the implementation. This library supports both cgroup v1 and cgroup v2. 

#### systemd
Systemd, the init daemon of most linux hosts, is also a cgroup driver which can use cgroup to manage processes. It provides a wrapped way to bind process and cgroup, so that we don't have to manually manage the cgroup files. For example:
```
systemctl set-property {worker_id}.service CPUQuota=20% # Bind process and cgroup automatically.
systemctl start {worker_id}.service
```

NOTE: The entire config options is [here](https://man7.org/linux/man-pages/man5/systemd.resource-control.5.html). And we can also use `StartTransientUnit` to create cgroups with worker processes simply. This is a [dbus](https://www.freedesktop.org/wiki/Software/systemd/dbus/) API and there is a [dbus-python](https://dbus.freedesktop.org/doc/dbus-python/) python lib that we can use.

#### Why we should support both cgroupfs and systemd
The cgroupfs and the systemd are two mainstream ways to manage cgroups. In [container technology](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/), this two ways are also supported. We should also support both of them.

The reason of using cgroupfs:
- Cgroupfs is a traditional and common way to manage cgroups. If your login user has the privilege of the cgroup file system, you can create the cgroups.
- In some environments, you couldn't use systemd, e.g. in a non-systemd based system or in a container which couldn't access the systemd service.

The reason of using systemd:
- Systemd is highly recommended by [runc](https://github.com/opencontainers/runc/blob/main/docs/cgroup-v2.md.) for cgroup v2.
- If we use cgroupfs in a systemd based system, there will be more than one component which simultaneously manages the cgroup tree.
- In the systemd [docs](https://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/), we can know that, in future, create/delete cgroups will be unavailable to userspace applications, unless done via systemd's APIs.
- The systemd has a good abstract API of cgroups. 

So, we should support both cgroupfs and systemd. And we will provide a environment variable named `RAY_CGROUP_MANAGER` which is used to change the cgroup manager in cluster level.

### Changes needed in Ray

1. Add a cgroup runtime environment plugin

The cgroup plugin is used to create or delete cgroups, and bind worker processes to cgroups. It is loaded in the Agent. 

The cgroup plugin should expose an abstract interface that can hide the differences between cgroupfs/systemd and cgroup v1/v2.

2. Generate the command line of cgroup.

Agent should generate the command line from the cgroup plugin. The command line will be appended to the `command_prefix` field of runtime env context. A command line sample is like:

```
mkdir /sys/fs/cgroup/{worker_id} && echo "200000 1000000" > /sys/fs/cgroup/foo/cpu.max && echo {pid} > /sys/fs/cgroup/foo/cgroup.procs
```

3. Start the worker process with the command line.

Currently, we use `setup_worker.py` to enforce the runtime environments. `setup_worker.py` will merge the `command_prefix` of runtime env context and real worker command. In the same way, the command about cgroup will be run with the worker process.


4. The enhancement of runtime environment pluggability

When worker processes die, we should delete the cgroup which is created for the processes. This work is only needed for cgroupfs because systemd can automatically delete the cgroup when the process dies.

So, we should make some changes to allow each runtime environment plugin to do some work when worker processes die.
- Add `worker_id` field to the existing PRC request `GetOrCreateRuntimeEnvRequest` and `DeleteRuntimeEnvIfPossibleRequest`. These two RPCs are used to create and delete runtime environments from the Raylet to the Agent.
- Add two methods, named `create_for_worker` and `delete_for_worker`, in the plugin interface. We already have two methods named `create` and `delete` in the interface. The difference is that, `delete` is called only when there is no worker using the runtime environment(The reference counting to be zero), but `delete_for_worker` is called every time when a process dies.

Another change is that we should support loading runtime env plugins when we start the Ray nodes. For example:
```
ray start --head --runtime-env-plugins="ray.experimental.cgroup_env_plugin:ray.experimental.test_env_plugin"
```
We will load the plugins and make configuration and validation. We should also add a new field `name` in the plugin interface to define the plugin name.

For more details, please see the [doc](https://docs.google.com/document/d/1x1JAHg7c0ewcOYwhhclbuW0B0UC7l92WFkF4Su0T-dk/edit?usp=sharing). 

## Compatibility, Deprecation, and Migration Plan

This proposal will not change any existing APIs and any default behaviors of Ray Core.

## Test Plan and Acceptance Criteria

We plan to benchmark the resources of worker processes:
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
  - (**Step 1**) The Raylet(Worker Pool) receives the publishing message of job starts.
  - (**Step 2**) The Raylet sends the RPC `GetOrCreateRuntimeEnv` to the Agent. 
  - (**Step 3**) The Agent sets up the `runtime_env`. For `resource_control`, the agent generates `command_prefix` in `runtime_env_context` to talk about how to enable the resource control.

When we create a `Task` or `Actor` with a `runtime_env`(or a inherited `runtime_env`), the main steps are:
  - (**Step 4**) The worker submits the task.
  - (**Step 5**) The `task_spec` is received by the Raylet after scheduling.
  - (**Step 6**) The Raylet sends the RPC `GetOrCreateRuntimeEnv` to the Agent.
  - (**Step 7**) The agent generates `command_prefix` in `runtime_env_context` and replies the RPC.
  - (**Step 8**) The Raylet starts the new worker process with the `runtime_env_context`.
  - (**Step 9**) The `setup_worker.py` sets up the `resource_control` by `command_prefix` for the new worker process.

### More references
- [Yarn NodeManagerCgroups](https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/NodeManagerCgroups.html)
