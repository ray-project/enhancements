# Ray Authentication with Kubernetes RBAC

## General Motivation

Ray v2.52.0 introduced support for token authentication, enabling Ray to enforce the use of a single, statically generated token in the authorization header for all requests to the Ray Dashboard and GCS server.

For Ray clusters running on Kubernetes, Ray can leverage Kubernetes RBAC to handle authentication and authorization. In most Kubernetes environments, Ray workloads and users already possess an identity, either directly through Kubernetes Service Accounts or via integrations such as webhook token authentication or OpenID Connect tokens. These integrations allow Kubernetes to interface with external IAM systems (e.g., GKE using Google IAM for access control).

Integrating Ray with Kubernetes RBAC allows Ray to benefit from the robust security features of Kubernetes RBAC and provides seamless authentication integration across major cloud providers and their IAM systems (e.g., GKE, EKS, AKS). This integration empowers platform teams to use familiar Kubernetes APIs for access control management and automated credential rotation.

This proposal details how Ray will be extended to delegate authentication and authorization to Kubernetes RBAC.

## Should this change be within `ray` or outside?

This change will affect both `ray` and `KubeRay`.

## Stewardship

### Owners

- @andrewsykim

### Reviewers

- @sampan-s-nayak
- @richo-anyscale
- @rueian
- @Future-Outlier

### Shepherd of the Proposal (should be a senior committer)

@edoakes

## Design and Architecture

To enable Kubernetes-based authentication and authorization, Ray will introduce a new authentication mode, configured via the environment variable `RAY_AUTH_MODE=k8s`. Unlike the existing token mode, which validates requests against a single static token on the Ray head, this new mode will perform two API calls to the Kubernetes API server:

* `TokenReview`: an API to authenticate a token to a known user.
* `SubjectAccessReview`: an API to check whether a user or group is authorized to perform an action.

![Ray Authentication with Kubernetes](ray_auth_with_k8s_architecture.png)

## Implementation Plan

### Authentication and Authorization Flow

Upon receiving a request containing a token, Ray will validate tokens by first calling the TokenReview API.
All TokenReview requests from Ray must specify the `ray.io` audience to prevent various forms of token misuse.
Below is an example a TokenReview request:

```yaml
apiVersion: authentication.k8s.io/v1
kind: TokenReview
spec:
  audiences:
  - ray.io
  token: a0AXooCgfzVabfasadftbNJ_4hl5556344534fZU0GsDlj...
```

If the token belongs to an authenticated user (as determined by Kubernetes), Ray will receive a response with a `TokenReview` status similar to the following:

```yaml
apiVersion: authentication.k8s.io/v1
kind: TokenReview
status:
  audiences: [...]
  authenticated: true
  user:
    groups:
    - system:authenticated
    - myteam@example.com
    - ray-admins
    - ray-users
    username: my-user@example.com
```

At this stage, token authentication is complete. For authorization, Ray will use the user and group information from the `TokenReview` status to construct a `SubjectAccessReview` request. An example `SubjectAccessReview` request follows:

```yaml
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  user: my-user@example.com
  resourceAttributes:
    verb: ray-user  # custom verb
    group: ray.io
    resource: rayclusters
    name: ray-cluster
    namespace: my-team
```

The `SubjectAccessReview` ensures that only authenticated users who are also granted RBAC access to the RayCluster with custom verbs like
`ray:read-user` and `ray:write-user` are allowed. How a user is granted access to this resource is up to the owner of the RayCluster or the cluster
admin of the Kubernetes cluster using standard Kubernetes RBAC APIs (ClusterRole, Role, RoleBinding, etc). The name and namespace of the Ray cluster
used in the SubjectAccessReview request will be stored in environment variables `RAY_CLUSTER_NAME` and `RAY_CLUSTER_NAMESPACE` configured by KubeRay.

Below is a sequence diagram illustrating the entire authentication and authorization flow:

![Ray Auth Flow](ray_auth_flow.png)

### Raylet Identity with Kubernetes Service Accounts

By default, the identity of the Raylet will be bound to the Service Account token of the Pod.
However, the Raylet will not use the default token in `/var/run/kubernetes.io/serviceaccount/token`.
Instead, a dedicated token in path `/var/run/ray.io/serviceaccount/token` will be mounted using
[serviceAccountToken projected volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/#serviceaccounttoken)
with a token audience dedicated for Ray. Below is an example of how this would be configured through KubeRay:

```yaml
spec:
  containers:
  - name: ray-head
    ...
    ...
    volumeMounts:
    - name: ray-token
      mountPath: "/var/run/ray.io/service-account"
      readOnly: true
  serviceAccountName: default
  volumes:
  - name: ray-token
    projected:
      sources:
      - serviceAccountToken:
          audience: ray.io
          expirationSeconds: 3600
          path: token
```

### Integration with External IAM

Most managed Kubernetes platforms provide integrations with external IAM systems. For example, on GKE, Google Cloud IAM and Kubernetes RBAC are
integrated to authorize users to perform actions. Google Cloud users can be referenced in Kubernetes RBAC objects to define and assign permissions.

When Ray is enabled to use Kubernetes for authn/authz, it will also enable Ray users to use their external identities to connect to their clusters.
For example, on GKE you can set `RAY_AUTH_TOKEN` to a token from gcloud as long as it specifies the intended audience:

```bash
$ export RAY_AUTH_TOKEN=$(gcloud auth print-identity-token --audiences=ray.io --impersonate-service-account="my-account@example.iam.gserviceaccount.com")

$ ray job submit ...
```

### Token caching in Ray

Ray will cache authenticated tokens with a default TTL of 5 minutes to avoid querying the Kubernetes API for every request.
Depending on user feedback, we may make the cache TTL configurable in the future.

### KubeRay enhancements

The authentication mode can be enabled in KubeRay using the following API:

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  ...
spec:
  authOptions:
    mode: k8s
```

Unlike in `token` mode, KubeRay will not generate Secrets containing tokens. Instead, it will create a RoleBinding that binds the Service Account used by the `RayCluster` to a ClusterRole, granting the `ray-user` verb for all `RayCluster`s in the namespace.

The KubeRay Helm chart will be updated to include a default ClusterRole that grants access to the `ray-user` custom verb:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ray-users
rules:
- apiGroups: ["ray.io"]
  resources: ["rayclusters"]
  verbs: ["ray:read", "ray:write"]   # custom verb
```

The KubeRay Helm chart will also contain a ClusterRoleBinding for the service account used by the KubeRay operator:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ray-users
subjects:
- kind: ServiceAccount
  name: kuberay-operator
  namespace: default
roleRef:
  kind: ClusterRole
  name: ray-users
  apiGroup: rbac.authorization.k8s.io
```

This ClusterRoleBinding grants the KubeRay operator access to every `RayCluster`. This access is required for `RayJob` and `RayService` custom resources, as KubeRay needs authenticated access to Ray clusters to submit jobs and manage Serve applications.

## (Optional) Follow-on Work

### Configurable cache TTL for tokens

We may explore optimizations for token caching and provide an option for users to configure the token cache TTL.
