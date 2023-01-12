## Summary - AWS Fleet support for ray autoscaler

### General Motivation

Today, AWS autoscaler requires developers to choose their EC2 instance types upfront and codify it in their [autoscaler config](https://github.com/ray-project/ray/blob/master/python/ray/autoscaler/aws/example-full.yaml#L73-L80). EC2 offers a lot of instances types across different families with varied pricing. Hence, developers can realize cost-savings if their workload does not have a strict dependency on a specific family of instances. [EC2 Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet.html) is an AWS offering that allows developers to launch a group of instances depending on various parameters like, maximum amount per hour they are willing to pay, supplement their primary on-demand capacity with spot capacity, specify maximum spot prices for each instance, choose from multiple allocation strategies for on-demand and spot capacity etc. This proposal outlines the work needed to support EC2 Fleet on ray autoscaler.

#### Key requirements

-   Autoscaler must offer existing functionalities and hence changes must be backward compatible.
-   Developers must be able to make use of the features including but not limited to following:
    -   save costs by combining on-demand capacity with spot capacity
    -   make use of different spot and on-demand allocation strategies like `price-capacity-optimized`, `diversified`, `lowest-price` etc. to scale up
    -   specify maximum spot prices for each instance to optimize their costs
    -   choose optimal instance pools based on availability depending on vCPU and memory requirements instead of having to choose instance type upfront
-   [Ray autoscaling monitor](https://github.com/ray-project/ray/blob/a03a141c296da065f333ea81445a1b9ad49c3d00/python/ray/autoscaler/_private/monitor.py), [CLI](https://github.com/ray-project/ray/blob/7b4b88b4082297d3790b9e542090228970708270/python/ray/autoscaler/_private/commands.py#L692), [standard autoscaler](https://github.com/ray-project/ray/blob/a03a141c296da065f333ea81445a1b9ad49c3d00/python/ray/autoscaler/_private/autoscaler.py), and placement groups must be able to provision nodes via EC2 fleet.
-   EC2 Fleet must not interfere with the autoscaler activities.
-   EC2 Fleet must be supported for both head and worker node types.

### Should this change be within `ray` or outside?

Yes. Specifically within AWS autoscaler.

## Stewardship

### Required Reviewers

-   TBD

### Shepherd of the Proposal (should be a senior committer)

-   TBD

## Design and Architecture

### Proposed Architecture

![Fleet REP](https://user-images.githubusercontent.com/8843998/211219167-eb18a917-c17b-46df-94ad-83aedd8c878b.png)

As described above, we will be adapting **node provider** to support instance creation via [create_fleet](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html#EC2.Client.create_fleet) API of EC2. The existing components that invoke **node provider** are only shown for clarity.

### How EC2 fleet works?

An EC2 Fleet contains configuration to launch a group of instances. There are [three types](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet-request-type.html) of fleet requests, namely, `request`, `maintain` and `instant`. Because of the constraints mentioned in the **requirements** section (i.e., EC2 must not interfere with autoscaler activities), we would only be able to make use of `instant` type. Unlike, `maintain` and `request`, `instant` type `create_fleet` creates a fleet entity synchronously in an AWS account which is simply a configuration. Below are couple of observations related to `instant` type EC2 fleets:

-   `instant` type fleets cannot be deleted without terminating instances it created.
-   directly terminating instances created by `instant` fleet using `terminate_instances` API does not affect the fleet state.
-   spot interruptions do not replace an instance created by `instant` fleet.
-   there is no limit to number of EC2 `instant` type fleets one can have in an account.

### How will EC2 fleets be created?

An EC2 fleet configuration will be encapsulated within a single node type from the autoscaler perspective. We aim to create a new EC2 `instant` fleet as part of `create_node` method that is pretty much the entry point for any scale-up request from any other component (per diagram above). Ray's autoscaler will make sure `max_workers` config is honoured as well as perform any bin-packing of the resources across node types. EC2 automatically tags the instances created this way with a tag named `aws:ec2:fleet-id`.

### How will the demand be converted into resources?

Autoscaling monitor receives several demands from the metrics like vCPUs, memory, gpus etc. The [resource_demand_scheduler.py](https://github.com/ray-project/ray/blob/master/python/ray/autoscaler/_private/resource_demand_scheduler.py) is responsible for converting the demand into number of workers per node type specified in the autoscaler config. However, it relies on the **node provider** to statically determine the CPU, memory, GPU and any custom resources. Hence, **node_provider** will determine CPU, memory and GPU from the `InstanceRequirements` and specified `InstanceType` parameters and aggregate them based on the low spec instance. Hence, autoscaler may end up spinning up more nodes than necessary which leads to overscaling. Autoscaler will also identify the idle nodes and remove them if required in the subsequent iterations. However, the [current behavior](https://github.com/ray-project/ray/blob/master/python/ray/autoscaler/_private/aws/node_provider.py#L403) also does not guarantee target capacity will reached. Autoscaling monitor loop will figure out any underscaling issue

### How will spot interruptions be handled?

Developers often choose to use spot instances if their application can tolerate faults while they could optimize for costs. When there is a spot interruption, GCS will fail to receive heartbeat from that node essentially marking that node as dead which could trigger a scale up signal from autoscaler. Fortunately, there is [no quota](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/fleet-quotas.html) for number of fleets with `instant` type which allows us to scale up unlimited number of times each with different fleet id.

### How will EC2 fleets clean up happen?

There will be no change in how the instances are terminated today. However, terminating all the instances in a fleet does not necessarily delete the fleet. The fleet entity continues to be in active state. Hence, a `post_process` step will be introduced after each autoscaler update which cleans up any unused active fleets. We cannot describe fleets of `instant` type via EC2 APIs, hence it must be stored in either GCS kv store or any in-memory storage accessible to autoscaler.

### How will caching stopped nodes work?

When stopped nodes are cached, they are reused next time the cluster is setup. There is no change in this behavior w.r.t EC2 fleets. Any changes in instance states within a fleet does not affect fleet state based on the POC. Hence, they have to be explicitly deleted as part of `post_process`.

### How will customer experience looks like?

The autoscaler config or the customer experience with respect to provisioning workers using EC2 `create_instances` API will not change. An optional config key `node_launch_method` (note the naming convention) will be introduced within the `node_config` that determines whether to use `create_fleet` API or `create_instances` API while provisioning EC2 instances. However, default value for this config key will be `create_instances`. Any other arguments within the `node_config` will be passed to the appropriate API as is. However, the request type will be overridden to `instant` as Ray autoscaler cannot allow EC2 fleets to interfere with the scaling.

```yaml
....
ray.worker.default:
    max_workers: 100
    node_config:
      node_launch_method: create_fleet
      OnDemandOptions:
        AllocationStrategy: lowest-price
      LaunchTemplateConfigs:
        - LaunchTemplateSpecification:
            LaunchTemplateName: RayClusterLaunchTemplate
            Version: $Latest
          Overrides:
            - InstanceType: r3.8xlarge
              ImageId: ami-04af5926cc5ad5248
            - InstanceType: r4.8xlarge
              ImageId: ami-04af5926cc5ad5248
            - InstanceType: r5.8xlarge
              ImageId: ami-04af5926cc5ad5248
      TargetCapacitySpecification:
        DefaultTargetCapacityType: 'on-demand'
....
```

### What are the known limitations?

-   The fleet request types `maintain` and `request` are not supported. Also, a hard limit on number of fleets of type `maintain` or `request` may block us from supporting that in future as well.
-   EC2 fleet cannot span different subnets from same availability zone. This is more of a limitation of EC2 fleets which could impact availability.
-   Autoscaler can scale up multiple times less than `upscaling_speed` when the `InstanceType` overrides are heterogeneous from instance size point of view. Hence, developers must be cautioned to follow best practices through proper documentation.

### Proposed code changes

-   The default version of `boto3` that is installed must be upgraded to latest i.e., `1.26.41` as `ImageId` override param may be missing in earlier version.
-   Update [config.py](https://github.com/ray-project/ray/blob/e464bf07af9f6513bf71156d1226885dde7a8f46/python/ray/autoscaler/_private/aws/config.py) to parse and update configuration related to fleets.
-   Update the [node_provider.py](https://github.com/ray-project/ray/blame/00d43d39f58f2de7bb7cd963450f7a763b928d10/python/ray/autoscaler/_private/aws/node_provider.py#L250) to create instances using EC2 fleet API like [this](https://github.com/ray-project/ray/commit/81942b4f8c8e9d9c6a037d068e559769e8a27a70).
-   EC2 does not delete a fleet when all of its instances are terminated. Hence, implement [post_process](https://github.com/ray-project/ray/blob/c51b0c9a5664e5c6df3d92f9093b56e61b48f514/python/ray/autoscaler/node_provider.py#L258) method for aws node provider to clean up any active fleets which has only terminated instances.
-   Add an example [autoscaler config](https://github.com/ray-project/ray/tree/master/python/ray/autoscaler/aws) documentation to help developers in utilizing the EC2 fleet functionality.
-   Validate the [launch config check logic](https://github.com/ray-project/ray/blob/a03a141c296da065f333ea81445a1b9ad49c3d00/python/ray/autoscaler/_private/autoscaler.py#L541) given the `node_config` will be different for EC2 fleet.
-   Update documentation to recommend developers to use instances with similar resource shape to avoid overscaling though EC2 fleet allows them to acquire instances at lower costs.
-   Update ray test suite to cover integration and unit tests.

## Compatibility, Deprecation, and Migration Plan

The changes are supposed to be backward compatible.

## Test Plan and Acceptance Criteria

### Test Plan

-   Setup ray head using EC2 fleets via CLI

-   Setup ray worker nodes using EC2 fleets via CLI

-   Setup ray cluster using the EC2 fleet and run varieties of ray applications with:

    -   task requesting n cpus
    -   task requesting n gpus
    -   actors requesting n cpus
    -   actors requesting n gpus
    -   tasks and actors requesting combination of cpu/gpu and custom resources.

-   Setup cluster with below outlined autoscaler config variations:

    -   fleet request contains InstanceRequirements overrides.
    -   fleet request contains InstanceType overrides.

### Acceptance Criteria

-   Ray application comply with resource (cpu / gpu / memory / custom) allocation.
-   InstanceRequirements, InstanceType parameters work as expected.
-   Less than 30% performance overhead.
