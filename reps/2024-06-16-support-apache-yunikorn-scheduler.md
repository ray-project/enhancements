# Support new batch scheduler option: Apache YuniKorn

## Motivation

The [Support batch scheduling and queueing](https://github.com/ray-project/kuberay/issues/213) allows Ray to integrate
easily with non-default Kubernetes scheduler, such as [Volcano](https://volcano.sh/). This proposal aims to add support for
another popular Kubernetes scheduler: [Apache YuniKorn](https://yunikorn.apache.org/). This provides Ray users with 
adds another option to leverage the [scheduling features](https://yunikorn.apache.org/docs/next/get_started/core_features)
YuniKorn offers.

## Existing state

### Current Batch Scheduler Option

Currently, the batch scheduler feature can be enabled using the `enable-batch-scheduler` boolean option.
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
However, once the `enable-batch-scheduler` option is set to `true`, the scheduler manager will attempt to load all
scheduler schemas by calling the `AddToScheme(scheme *runtime.Scheme)` function implemented by each scheduler plugin.
This loads all the schemas, including CRDs defined in the implementation. Since only the `VolcanoBatchScheduler`
is currently implemented, it always loads Volcano's `PodGroup` CRD in the controller runtime,
requiring the installation of Volcano CRDs even when enabling other scheduler options.

## Proposed changes

### Targeted scheduler option

To support Apache YuniKorn, and more importantly, to other schedulers in Ray.
We propose to add an option to explicitly set the scheduler name, i.e `--batch-scheduler-name`.
Option syntax: `--batch-scheduler-name [SupportedSchedulerName]`, for example:

Option Syntax:

```shell
# 1. use volcano
--enable-batch-scheduler=true
--batch-scheduler-name=volcano

# 2. use yunikorn
--enable-batch-scheduler=true
--batch-scheduler-name=yunikorn

# 3. use default Kubernetes scheduler, or do not turn on the feature flag
--enable-batch-scheduler=true
--batch-scheduler-name=default
```

When a scheduler name is specified, the scheduler manager will only load the configured scheduler plugin,
ensuring only necessary resources are loaded in the controller runtime.

The option `--batch-scheduler-name` accepts a single scheduler name as the value, and the value must be `default`,
`volcano` and `yunikorn` (before other scheduler plugins introduced). If an unrecognized scheduler name is provided,
the controller will fail to start with an error indicating that the scheduler plugin is not found.

### Deprecate "ray.io/scheduler-name"

With the scheduler name is specified in the operator startup options, there is no need to set `ray.io/scheduler-name`
in `RayJob` or `RayCluster` CRs. This option should be marked as deprecated and eventually removed.

### Compatibility

When this change is introduced, existing users who use `--enable-batch-scheduler` with the Volcano scheduler
must explicitly add `--batch-scheduler-name=volcano`. Otherwise, the controller will default to the Kubernetes scheduler.
This is an incompatible change and must be carefully documented to mitigate potential impact.

### YuniKorn scheduler plugin behavior

The YuniKorn scheduler plugin will support both `RayJob` and `RayCluster` resources. The integration will be lightweight,
as YuniKorn does not require new CRDs. The plugin will add labels to the Ray pods, and the YuniKorn scheduler will
schedule Ray pods based on these labels.

To enable yunikorn scheduler, set the following options when starting the KubeRay operator:

```shell
--enable-batch-scheduler=true
--batch-scheduler-name=yunikorn
```

if submitting a `RayCluster`, add `yunikorn.apache.org/queue-name` and `yunikorn.apache.org/application-id` to the labels.

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  labels:
    ray.io/scheduler-name: yunikorn
    yunikorn.apache.org/queue-name: root.abc
    yunikorn.apache.org/application-id: rayjob-sample-ltpjh 
```

The `RayJob` will be submitted to `root.abc` queue and scheduled by the yunikorn scheduler. The `RayJob` will be
recognized as a "job" with ID "rayjob-sample-ltpjh".

If submitting a `RayJob`, provide only the scheduler name and queue name:

```yaml
apiVersion: ray.io/v1
kind: RayJob
metadata:
  name: rayjob-example
  namespace: my-namespace
  labels:
    ray.io/scheduler-name: yunikorn
    yunikorn.apache.org/queue-name: root.abc
```

when the Ray job is submitted to the cluster, the Ray operator will create the following RayJob CR:

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  labels:
    # the controller populates the annotations with RayJob annotations
    ray.io/scheduler-name: yunikorn
    yunikorn.apache.org/queue-name: root.abc
    # the same job ID generated by the controller
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


