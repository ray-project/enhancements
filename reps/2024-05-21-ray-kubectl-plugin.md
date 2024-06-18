# Ray Kubectl Plugin

This document introduces a kubectl plugin designed to enhance the interaction with KubeRay resources, simplifying the experience of utilizing Ray on Kubernetes.
The primary objective of this plugin is to provide a user-friendly interface, enabling users to leverage the benefits of Ray on Kubernetes without needing extensive knowledge of Kubernetes concepts and tools.

## Motivation

Today it is incredibly challenging for data scientists and AI researchers unfamiliar with Kubernetes to start using Ray on Kubernetes.
However, running Ray on Kubernetes is advantageous or necessary in certain environments. These users struggle with KubeRay for a variety of reasons:
1. **Unfamiliarity with Kubernetes API and Best Practices**: Users new to Kubernetes may struggle to operate Ray clusters using Kubernetes concepts like Pods and Services due to unfamiliarity with the Kubernetes API and best practices.
2. **Complex Kubernetes Networking**: Kubernetes networking can be daunting for beginners. Understanding how to use kubectl port-forward or externally exposed Services to connect to Ray clusters can be challenging without prior experience.
3. **Construction of Kubernetes YAML Manifests**: Creating YAML manifests for RayCluster, RayJob, and RayService can be challenging for those not experienced with using Kubernetes custom resources.

For novice Kubernetes users, an intuitive CLI experience can drastically enhance their onboarding journey. A kubectl plugin is proposed since it can seamlessly integrate with the rest of the existing kubectl surface as needed.
The primary goal of this plugin should be user-friendliness and a smooth progression from zero to “something good enough”. For more advanced scenarios and scaling requirements, users can opt to manage KubeRay custom resources
independently as many users do today.

More simply, the Ray Kubectl Plugin simplifies the management of Ray clusters by eliminating the need for users to handle complex YAML configurations.

Why a kubectl plugin instead of the existing kuberay CLI?
* **Convenience / ease of use**: Kubectl plugins are designed to work within the existing kubectl framework. They inherit the user's current authentication, context, and cluster metadata, eliminating the need for additional setup.
* **Seamless kubectl integration**: Users can effortlessly switch between basic kubectl subcommands and the Ray plugin using a single command-line tool.
* **Accessibility**: The Kuberay CLI needs the KubeRay API server to function. However, the majority of clusters using KubeRay don't run the KubeRay API server, which limits the accessibility of the CLI.

## User Journey

### Create / Manage a Ray cluster

```
$ kubectl ray cluster create my-cluster --ray-version=2.9.3
    --worker-replicas=10 --worker-cpus=8 -worker-memory=32GB --worker-gpus=2
```

```
kubectl ray cluster scale default-worker-group --cluster my-cluster --replicas=10
```

```
kubectl ray cluster worker-group create cpu-pool --cluster my-cluster -worker-replicas=10
    --worker-cpus=8 -worker-memory=32GB

```

```
kubectl ray cluster delete my-cluster
```

### Ray Job Submissions (using RayJob)

```
$ kubectl ray job submit --cluster my-cluster --image=image:tag -- python job.py
```

### Ray Session - Interactive Client / Dashboard

```
$ kubectl ray cluster session my-cluster
Connecting...
Ray Interactive Client Session started on port 10001...
Ray Dashboard Session started on port 8265...
```

### Ray Dashboard

```
$ kubectl ray cluster dashboard my-cluster
Opening browser session to Ray dashboard ...
```

### Ray Logs

```
$ kubectl ray cluster logs my-cluster --out-dir ./log-dir
Downloading Ray logs to ./log-dir ...
```

## Implementation Details

The kubectl plugin will be developed in the cli directory, replacing the current KubeRay CLI. While the kubectl plugin will overlap with the existing KubeRay CLI in some ways (especially in managing KubeRay cluster), it will go further by enhancing day-to-day operations with Ray clusters. These enhancements include authenticating to clusters, establishing local sessions, submitting jobs, and scaling clusters. In addition, the kubectl plugin will not depend on the KubeRay API server, making it viable for a larger audience of KubeRay users.

The CLI will extend kubectl using kubectl’s plugin mechanism. See [Extend kubectl with plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/) for more details.

MVP Scope:
* `kubectl ray cluster get|list`
* `kubectl ray cluster scale`
* `kubectl ray cluster session`
* `kubectl ray cluster dashboard`
* `kubectl ray cluster logs`

Future Scope:
* `kubectl ray cluster create|update|delete`
* `kubectl ray create cluster –provider=gke|eks|aks|etc`
    * Support for adding provider specific YAML like GCSFuse/EFS mounts, loadbalancers, etc
* `kubectl ray job get|list`
* `kubectl ray job submit`
