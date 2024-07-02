# Kubernetes Native Ray Authentication

Ray, in its default configuration, lacks robust security and authentication measures.
Its deployment on Kubernetes provides some level of protection by leveraging RBAC for access control of RayCluster resources.
However, once provisioned, a RayCluster remains vulnerable to unauthorized access from anyone with network connectivity to the Ray head node.

This proposal introduces a Kubernetes aware sidecar proxy for authenticating external access to the Ray head.
It will leverage the existing Kubernetes authentication system to grant tokens used to securely access Ray head endpoints.
This simplifies Ray authentication by centralizing management of tokens and users within Kubernetes.

## Authentication Scope

While there is a large surface area of a Ray cluster that could benefit from authenticated access, this proposal focuses specifically on securing Ray head endpoints that are frequently accessed externally.
This includes enforcing authentication for both the dashboard server and the interactive client server. Securing access from internal clients like the Raylet to the GCS server is not addressed in this proposal
as network policies are typically sufficient to protect this communication.

## Sidecar Authentication Proxy

To enforce authenticated access to specific Ray head ports, the KubeRay operator will deploy a sidecar container alongside the Ray head pod.
This sidecar will function as a reverse proxy, validating authentication tokens for requests to the dashboard port (8265) and the interactive client port (10001).
Traffic to other ports will pass through the sidecar proxy without requiring authentication.

The KubeRay operator will be changed in the following ways:
1. The Ray head container will bind to localhost only
2. A reverse proxy sidecar will run alongside the Ray head container
3. A ServiceAccount is created per RayCluster resource

The sidecar ensures only requests with the correct authorization tokens can access the Ray head. More details on how tokens are authenticated below.
For the MVP, the sidecar will be implemented using Envoy configured with an [External Authorizer](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter).

## Access Control with TokenReview API

The sidecar authenticator will use the TokenReview API to authenticate tokens passed into the authorization headers of each request.
Tokens can be created from Kubernetes ServiceAccounts or an external identity provider as long as the Kubernetes API Server is configured with an external authorizer.
Tokens from Kubernetes Service Accounts are required to specify a unique audience for the cluster. For now the placeholder audience is "ray.io/cluster/<cluster-name>".
Each RayCluster will be provisioned a default ServiceAccount that KubeRay Operator will use when authenticating to the Ray head (specifically for RayJob and RayService).
Users can use the default ServiceAccount in the absence of external identity providers, but using external identity providers is strongly recommended.

Example of a TokenReview request that contains a token:
```
apiVersion: authentication.k8s.io/v1
kind: TokenReview
spec:
  token: <your_token_here>
  audiences:
  - "ray.io/cluster/<cluster-name>"
```

Example of a TokenReview response with user information about the token:
```
apiVersion: authentication.k8s.io/v1
kind: TokenReview
status:
  authenticated: true
  user:
    username: system:serviceaccount:default:ray-cluster
    uid: <some_uid>
    groups: ["system:serviceaccounts", "system:serviceaccounts:default"]
  audiences:
  - "ray.io/cluster/<cluster-name>"
```

## User to Ray Head Authentication

When submitting jobs using the Ray job CLI:
```
export RAY_JOB_HEADERS="Authorization: Bearer $(kubectl create token ray-cluster)"
OR
export RAY_JOB_HEADERS="Authorization: Bearer $(gcloud auth print-access-token)" # example external identity provider

ray job submit...
```

When using Ray interactive client:
```
export RAY_CLUSTER_TOKEN=$(kubectl create token ray-cluster)
OR
export RAY_CLUSTER_TOKEN=$(gcloud auth print-access-token) # example external identity provider
------------------------------------------------------------

import os
import ray
import base64

def get_metadata():
    """
    Get authentication header metadata for ray.util.connect
    """
    headers = {"Authorization": f"Bearer {os.environ["RAY_CLUSTER_TOKEN"]}"}
    return [(key.lower(), value) for key, value in headers.items()]

ray.init(
    "ray-cluster:443",
    metadata=get_metadata(),
    secure=True,
)
```

## RayCluster API Changes

Enabling authentication should be optional per RayCluster. Two new fields `enableAuthentication` and `authenticationOptions` will be added to the RayCluster spec to enable this capability and configure allowed principals.
If no allowed principals are specified, it will default to a ServiceAccount with the same namespace and name as the RayCluster.

```
apiVersion: ray.io/v1
kind: RayCluster
metdata:
  ...
spec:
  enableAuthentication: true
  authenticationOptions:
    allowedPrincipals:
    - my-user@example.com   # example user
    - system:serviceaccount:default:ray-cluster  # example service account
    - system:authenticated  # example group
```

## Dynamic Access Control with Kubernetes RBAC and the SubjectAccessReview API

Beyond token authentication via the TokenReview API, Kubernetes Role-Based Access Control (RBAC) provides a dynamic mechanism to manage which principals (users or service accounts)
have access to Ray clusters. This can be achieved by introducing a custom verb and leveraging the SubjectAccessReview API.

A custom verb `admin` can be used in Roles that reference the `rayclusters` resource:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ray-admins
  namespace: my-team
rules:
- apiGroups: ["ray.io"]
  resources: ["rayclusters"]
  verbs: ["admin"]
```

Role bindings to this role can grant users access to the Ray cluster. The auth proxy can use the `SubjectAccessReview` API to verify if
the authenticated user also has admin access to the targetted Ray cluster.

Below is an example `SubjectAccessReview` request that would be sent from the auth proxy:
```
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  user: userA
  resourceAttributes:
    verb: admin
    group: ray.io
    resource: rayclusters
    name: ray-cluster
    namespace: my-team
```

If the authenticated user has access to the `admin` verb for the Ray cluster resource, the `SubjectAccessReview` request returns an `allowed` status
and the auth proxy grants access to the authenticated user.

## Risks

Enabling the authentication proxy puts the Kubernetes control plane in the critical path for accessing the Ray head. This is a trade-off users should consider when enabling this capability.
To mitigate this, we may consider implementing an in-memory cache of access control rules in the sidecar proxy so that every request to the sidecar does not require a TokenReview or SubjectAccessReview request.
