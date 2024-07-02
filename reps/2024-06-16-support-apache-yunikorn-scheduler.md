# Support new batch scheduler option: Apache YuniKorn

## Motivation

The [Support batch scheduling and queueing](https://github.com/ray-project/kuberay/issues/213) allows Ray to integrate
easily with non-default Kubernetes scheduler, such as [Volcano](https://volcano.sh/). This proposal aims to add support for
another popular Kubernetes scheduler: [Apache YuniKorn](https://yunikorn.apache.org/). This provides Ray users with 
another option to leverage the [scheduling features](https://yunikorn.apache.org/docs/next/get_started/core_features)
YuniKorn offers.

## Existing state

### Current Batch Scheduler Option

Currently, the batch scheduler feature can be enabled using the `--enable-batch-scheduler` boolean option.
When set to `true`, the operator is started with the following Helm chart value override:

```shell
--set batchScheduler.enabled=true
```

The scheduler manager in the KubeRay operator will initiate the scheduler plugins. The framework provides hooks for
each scheduler implementation to inject custom resources and modify pod metadata accordingly. When a Ray cluster is
created, the framework calls the appropriate scheduler plugin functions based on the scheduler name provided
by `ray.io/scheduler-name`.

### Limitations

The framework is designed to be scheduler-agnostic and provides general hooks for supporting different scheduler options.
However, once the `--enable-batch-scheduler` option is set to `true`, the scheduler manager will attempt to load all
scheduler schemas by calling the `AddToScheme(scheme *runtime.Scheme)` function implemented by each scheduler plugin.
This loads all the schemas, including CRDs defined in the implementation. Since only the `VolcanoBatchScheduler`
is currently implemented, it always loads Volcano's `PodGroup` CRD in the controller runtime,
requiring the installation of Volcano CRDs even when enabling other scheduler options.

## Proposed changes

### Targeted scheduler option

To support Apache YuniKorn, and more importantly, to support other schedulers in Ray.
We propose to add an option to explicitly set the scheduler name, i.e `--batch-scheduler`.
Option syntax: `--batch-scheduler [SupportedSchedulerName]`, for example:

Option Syntax:

```shell
# use volcano
--batch-scheduler=volcano

# use yunikorn
--batch-scheduler=yunikorn
```

When a scheduler name is specified, the scheduler manager will only load the configured scheduler plugin,
ensuring only necessary resources are loaded in the controller runtime.

The option `--batch-scheduler` accepts a single scheduler name as the value, and the value must be `volcano`
or `yunikorn` (before other scheduler plugins introduced). If an unrecognized scheduler name is provided,
the controller will fail to start with an error indicating that the scheduler plugin is not found.

### Deprecate "ray.io/scheduler-name"

With the scheduler name is specified in the operator startup options, there is no need to set `ray.io/scheduler-name`
in `RayJob` or `RayCluster` CRs. This option should be marked as deprecated and eventually removed.

### Compatibility

To maintain backwards compatibility, the `--enable-batch-scheduler` option will remain supported for a few more
releases. However, it will be marked as deprecated, and users are encouraged to switch to the new option.
Below are the scenarios how these options can be used:

```shell
# case 1:
#  use volcano
--enable-batch-scheduler=true

# case 2:
#  use volcano
--batch-scheduler=volcano

# case 3:
#  use yunikorn
--batch-scheduler=yunikorn

# case 4:
#  invalid options, error message: do not use two options together
#  for simplicity, only one of these 2 options should be used
--enable-batch-scheduler=true
--batch-scheduler=volcano|yunikorn
```

### YuniKorn scheduler plugin behavior

The yunikorn scheduler plugin will support both `RayCluster` and `RayJob` resources. The integration will be lightweight,
as yunikorn does not require new CRDs. The plugin will add labels to the Ray pods, and the yunikorn scheduler will
schedule Ray pods based on these labels.

To enable yunikorn scheduler, set the following options when starting the KubeRay operator:

```shell
--batch-scheduler=yunikorn
```

if submitting a `RayCluster`, add `yunikorn.apache.org/queue-name` and `yunikorn.apache.org/application-id` to the labels.

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  labels:
    yunikorn.apache.org/queue-name: root.abc
    yunikorn.apache.org/application-id: rayjob-sample-ltpjh 
```

The `RayCluster` will be submitted to `root.abc` queue and scheduled by the yunikorn scheduler. The `RayCluster` will be
recognized as an "application" with ID "rayjob-sample-ltpjh".

If submitting a `RayJob`, provide only the queue name:

```yaml
apiVersion: ray.io/v1
kind: RayJob
metadata:
  name: rayjob-example
  namespace: my-namespace
  labels:
    yunikorn.apache.org/queue-name: root.abc
```

when the Ray job is submitted to the cluster, the Ray operator will create the following `RayCluster` CR:

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  labels:
    # RayCluster inherits the labels from RayJob
    yunikorn.apache.org/queue-name: root.abc
    # the same job ID defined in the RayJob spec, or generated by the controller 
    yunikorn.apache.org/application-id: rayjob-sample-ltpjh 
```

### YuniKorn scheduler plugin details

The YuniKorn scheduler plugin looks for relevant labels in the RayCluster and populates the following labels
to all the pods created by the RayCluster CR:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app.kubernetes.io/created-by: kuberay-operator
    app.kubernetes.io/name: kuberay
    # value is taken from RayCluster CR label: "yunikorn.apache.org/application-id"
    applicationId: rayjob-sample-ltpjh
    # value is taken from RayCluster CR label: "yunikorn.apache.org/queue"
    queue: root.abc
spec:
  schedulerName: yunikorn
```
Details about the meaning of these labels can be found in this
[doc](https://yunikorn.apache.org/docs/user_guide/labels_and_annotations_in_yunikorn#labels-and-annotations-in-yunikorn).
YuniKorn will recognize all these pods as part of the same application "rayjob-sample-ltpjh" and apply the
scheduling features accordingly.


