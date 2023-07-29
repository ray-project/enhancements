
## Summary - Serve gRPC Proxy
Improve Ray Serve GA on gRPC by offering a gRPC proxy to the users. The current
experimental gRPC ingress is a direct ingress to the replica. It lacks features we
built into HTTP proxy to keep Serve robust and generally available. This REP
describes how we can provide more performant gRPC service while reusing the GA
features we built with HTTP proxy.

### General Motivation
gRPC is generally a more performant way to talk to services. It utilizes the
protobuf to serialize requests resulting in smaller payloads and faster
communication. There are also customers who’s asking for this feature
(e.g. SecondDinner) to be able to serve faster.

There is an existing [Experimental gRPC Ingress](https://docs.ray.io/en/latest/serve/advanced-guides/direct-ingress.html)
feature we built to allow users to talk to their deployments directly. However,
this bypasses all the GA features we built for HTTP such as metrics, prevent
downscale request dropping, timeout, routing, model multiplexing…etc.

We have two options to bring GA to Serve’s gRPC feature. One is to reimplement
everything we built for HTTP into the existing gRPC ingress. Another is to
refactor and reuse most of existing code and offer a gRPC proxy for clients to
connect. If we chose option one, it will incur a lot of maintenance overhead
for any future features we add to the proxy. It’s also hard to keep parity
between the two implementations. Thus, exploring gRPC proxy is preferred for
long term maintenance and to offer high performance.


### Should this change be within `ray` or outside?
This feature will work with the changes within `ray` and within Serve. OSS users
will be able to start a gRPC service without changes from outside. However, platforms
running Ray Serve clusters will need to add new ALB entry and listeners to
the new gRPC port to serve with gRPC.


## Stewardship
### Required Reviewers
@edoakes, @ericl(?), @pcmoritz(?)

### Shepherd of the Proposal (should be a senior committer)

@edoakes

## Design and Architecture

### Example code
#### Ray Serve Protobuf API
```
message RayServeRequest {
  google.protobuf.Any user_request = 1;
  string application = 2;
  string request_id = 3;  // optional, feature parity w/ http proxy
  string multiplexed_model_id = 4;  // optional, feature parity w/ http proxy
}

message RayServeResponse {
  google.protobuf.Any user_response = 1;
  string request_id = 2;  // feature parity w/ http proxy
}

service RayServeService {
  rpc Predict(RayServeRequest) returns (RayServeResponse);
  rpc PredictStreaming(RayServeRequest) returns (stream RayServeResponse);
}
```

#### Unary gRPC example
```
# test_deployment.py

from ray import serve

# Users need to include their custom message type which will be embedded in the request.
from ray.serve.generated.serve_pb2 import UserDefinedMessage, UserDefinedResponse

@serve.deployment
class GrpcDeployment:
    def __call__(self, user_message: UserDefinedMessage) -> UserDefinedResponse:
        greeting = f"Hello {user_message.name} from {user_message.foo}"
        num_x2 = user_message.num * 2
        user_response = UserDefinedResponse(
            greeting=greeting,
            num_x2=num_x2,
        )
        return user_response


g = GrpcDeployment.options(name="grpc-deployment").bind()
```

```
# test_client.py

import grpc
from google.protobuf.any_pb2 import Any

from ray.serve.generated.serve_pb2_grpc import RayServeServiceStub

# Users need to include & build the Ray Serve protobuf so they can send requests
# via the request wrapper proto.
from ray.serve.generated.serve_pb2 import RayServeRequest

# Users also need to include their custom message type which will be embedded in
# the request.
from ray.serve.generated.serve_pb2 import UserDefinedMessage, UserDefinedResponse


channel = grpc.insecure_channel("localhost:9000")
stub = RayServeServiceStub(channel)

test_in = UserDefinedMessage(
    name="genesu",
    num=88,
    foo="bar",
)
user_request = Any()
user_request.Pack(test_in)  # Serialize the test_in to Any type
response = stub.Predict(
    RayServeRequest(
        application="default_grpc-deployment",
        user_request=user_request,
        # request_id="123",  # optional, feature parity w/ http proxy
        # multiplexed_model_id="123",  # optional, feature parity w/ http proxy
    )
)  # response is a type RayServeResponse
print("Output type:", type(response))
print("Full output:", response)
print("request_id:", response.request_id)

test_out = UserDefinedResponse()
response.user_response.Unpack(test_out)  # Deserialize the user_response to custom type
print("Output greeting field:", test_out.greeting)
print("Output num_x2 field:", test_out.num_x2)
```

#### Streaming gRPC example
```
# test_deployment.py

import time
from typing import Generator

from ray import serve
# Users need to include their custom message type which will be embedded in the request.
from ray.serve.generated.serve_pb2 import UserDefinedMessage, UserDefinedResponse


@serve.deployment
class GrpcDeploymentStreamingResponse:
  def __call__(self, user_request: UserDefinedMessage) -> Generator[UserDefinedResponse, None, None]:
    for i in range(10):
      greeting = f"{i}: Hello {user_request.name} from {user_request.foo}"
      num_x2 = user_request.num * 2 + i
      user_response = UserDefinedResponse(
        greeting=greeting,
        num_x2=num_x2,
      )
      yield ouser_response

      time.sleep(0.1)


g3 = GrpcDeploymentStreamingResponse.options(
  name="grpc-deployment-streaming-response"
).bind()
```

```
# test_client.py

import grpc

from ray.serve.generated.serve_pb2_grpc import RayServeServiceStub

# Users need to include & build the Ray Serve protobuf so they can send requests
# via the request wrapper proto.
from ray.serve.generated.serve_pb2 import RayServeRequest

# Users also need to include their custom message type which will be embedded in
# the request.
from ray.serve.generated.serve_pb2 import UserDefinedMessage, UserDefinedResponse


channel = grpc.insecure_channel("localhost:9000")
stub = RayServeServiceStub(channel)

test_in = UserDefinedMessage(
  name="genesu",
  num=88,
  foo="bar",
)
user_request = Any()
user_request.Pack(test_in)  # Serialize the test_in to Any type
responses = stub.PredictStreaming(
  RayServeRequest(
    application="default_grpc-deployment-streaming-response",
    user_request=user_request,
    # request_id="123",  # optional, feature parity w/ http proxy
    # multiplexed_model_id="123",  # optional, feature parity w/ http proxy
  )
)  # response is a generator of type RayServeResponse
for response in responses:
  print("Output type:", type(response))
  print("Full output:", response)
  print("request_id:", response.request_id)

  # Deserialize the user_response to custom type
  test_out = UserDefinedResponse()
  response.user_response.Unpack(test_out)
  print("Output greeting field:", test_out.greeting)
  print("Output num_x2 field:", test_out.num_x2)

```

### Design Diagram
![grpc_proxy_design](https://docs.google.com/drawings/d/e/2PACX-1vQdFOFoHpCYXtevujBg2YS2iaqpnnaPoGgEbdTlxKWhcmry6z9KQCOK1Fk00hQh76poNTnEiIyk4sRV/pub?w=1481&h=1047)
- `GenericProxy` is created that includes common variables and methods used by both
HTTP and gRPC proxies
  - `proxy_request()` method is refactored as an entrypoint for both protocols to use.
    It contains all the existing GA logics we implemented for HTTP proxy previously.
    Input will be a new `ServeRequest` type and output will be a new `ServeResponse`
    type so any protocol can share the same interface
- `HTTPProxy` is a subclass from `GenericProxy` includes specific implementations of
  - Unicorn entrypoint that wraps ASGI input objects (`Scope`, `Receive`, `Send`)
    into a `ServeRequest` object to use in shared `proxy_request()` method on
    the `GenericProxy`
  - Setup request context and handle based on the ASGI `Scope` (headers)
  - Methods to process request and consume and return the responses in ASGI
    `Send` object
- `GRPCProxy` is a subclass from `GenericProxy` includes specific implementations of
  - gRPC entrypoints one for unary and another for streaming. gRPC requires a
    predefined schema (input and output) for each method. The only difference between
    these two methods are the return type being a unary object or stream object.
    These two methods also wrap the gRPC input objects into a `ServeRequest` object
    to use in shared `proxy_request()` method on the `GenericProxy`
  - Setup request context and handle based on the input protobuf `RayServeRequest`
  - Methods to process request and consume and return the response as
    protobuf `RayServeResponse` object
- `RayServeHandle`, `Router`, and `Scheduler` all kept unchanged except
   added a new flag `serve_grpc_request` on the `HandleOptions` to signal to
   the replicas when it’s a gRPC request
- `Replica` handling HTTP requests all continue to use existing methods
  - New methods are added to handle requests in gRPC protocol
  - New if-statement is added to use gRPC methods when the `serve_grpc_request`
    flag is set true

![serve_data_flow_chart](https://docs.google.com/drawings/d/e/2PACX-1vTtg5xhPRvzghhNuOsFCPCllrWWZwVSqEGVpL3xd3ggTiFSKquW4x0sEmEjT3hsDvijmEo0ZOTiTVJO/pub?w=1527&h=866)
- HTTP clients continue to send requests through `HTTPProxy` using ASGI protocol.
  `HTTPProxy` will return the response in the ASGI `Send` object
- gRPC clients send predefined protobuf `RayServeRequest` to the `GRPCProxy`.
  `GRPCProxy` will return back a predefined protobuf `RayServeResponse` object.
  - `RayServeRequest` contains `user_request` as Any protobuf to pass to the replicas,
    `application` for which application to route to, `request_id` for tracking
    the request, and `multiplexed_model_id` for doing model multiplexing
 - `RayServeResponse` contains `user_response` as Any protobuf to return to the client
   and `request_id` for tracking the request
- The rest are existing code besides added a new `GRPCRequest` object to be used
  in place of `StreamingHTTPRequest` object in `RayServeHandle` and `Router`

### Change Details 
You can find the prototype PR here: [ray-project/ray#37310](https://github.com/ray-project/ray/pull/37310)

#### Proxy
- `GenericProxy` is refactored from `HTTPProxy`. It includes only the common variables
  and methods used by both proxies. It served as the parent class for both `HTTPProxy`
  and `GRPCProxy` to inherit from.
  - `proxy_request()` method is refactored as an entrypoint for both protocols to use.
    It contains all the existing GA logics we implemented for HTTP proxy previously.
    Input will be a new `ServeRequest` type and output will be a new `ServeResponse`
    type so any protocol can share the same interface
  - `proxy_name()` required to implement on the child class to return the name of
    the proxy used to populate metrics with the correct names
  - `setup_request_context_and_handle()` required to implement on the child class to
    setup the request context and handle based on the input objects
  - `send_request_to_replica_streaming()` required to implement on the child class to
    handle the response and return back to the user
- `HTTPPRoxy` is a subclass from `GenericProxy`. All the existing ASGI related
  methods are also moved in here
  - `__call__()` is still the entrypoint for HTTP requests. It wraps the ASGI
    input objects (`Scope`, `Receive`, `Send`) into a `ServeRequest` object and
    calls `proxy_request()`
  - `setup_request_context_and_handle()` is based on the ASGI `Scope` wrapped in
    the `ServeRequest` object
  - `send_request_to_replica_streaming()` does not need to change
- `GRPCProxy` is a subclass from `GenericProxy`.
  - `Predict()` is the entrypoint for unary gRPC requests. It takes a protobuf
    `RayServeRequest` object and returns a protobuf `RayServeResponse` object.
    It wraps the input into a `ServeRequest` object and calls `proxy_request()`
  - `PredictStreaming()` is the entrypoint for streaming gRPC requests. It takes
    a protobuf `RayServeRequest` object and returns a generator of protobuf
    `RayServeResponse` objects. It wraps the input into a `ServeRequest` object
    and calls `proxy_request()`
  - `setup_request_context_and_handle` is based on the input protobuf `RayServeRequest`
    `send_request_to_replica_streaming()` will handle request and return the
    response as protobuf `RayServeResponse` object or yield the result based on
    the stream flag
- `HTTPProxyActor` will be changed to start both `HTTPProxy` and `GRPCProxy`.
  We have the choice to always start `GRPCProxy` on the side of `HTTPProxy` or
  setup with [GRPCConfigs](#GRPCConfigs) follow up
- `LongestPrefixRouter` to add a new method `match_target()` to help gRPC look up for
  the routes from the application name

#### Handle and Router
New flag `serve_grpc_request` is added to the `HandleOptions` and to the
`RequestMetadata` to signal to the replicas when it’s a gRPC request. This flag
is boolean defaulting to false. It's only set when `GRPCProxy` is setting up the
handle.

#### Replica
- `call_user_method_grpc_unary()` is added to handle unary gRPC requests
- `call_user_method_with_grpc_stream()` is added to handle streaming gRPC requests
- A new if-statement is added to use gRPC methods when the `serve_grpc_request`
  flag is set true in `handle_request()` and in `handle_request_streaming()`

#### Data Models
- `GRPCRequest` is added to replace `StreamingHTTPRequest` in `RayServeHandle`
  and `Router` to be used for gRPC requests. It contains a `grpc_user_request`
  field to store the byte input for the replica and a `grpc_proxy_handle` field
  to store the ray serve handle to be used for gRPC requests.
- `RayServeRequest` is a protobuf object to be used for gRPC requests. It contains
  `user_request` as Any protobuf to pass to the replicas, `application` for which
  application to route to, `request_id` for tracking the request, and
  `multiplexed_model_id` for doing model multiplexing
- `RayServeResponse` is a protobuf object to be used for gRPC requests. It contains
  `user_response` as Any protobuf to return to the client and `request_id` for
  tracking the request
- `RayServeService` as protobuf service definition. It contains `Predict()` and
  `PredictStreaming()` methods to be used for gRPC requests
  - `Predict()` takes a protobuf `RayServeRequest` object and returns a protobuf
    `RayServeResponse` object
  - `PredictStreaming()` takes a protobuf `RayServeRequest` object and returns
    a stream of protobuf `RayServeResponse` objects
- `ServeRequest` is a new data model served as the input to `proxy_request()`
  method.
- `ASGIServeRequest` is a subclass of `ServeRequest` used by `HTTPProxy`.
  It contains the implementations to extract attributes from ASGI input objects for
  sending requests to the replica
- `GRPCServeRequest` is a subclass of `ServeRequest` used by `GRPCProxy`.
  It contains the implementations to extract attributes from gRPC input objects
  for sending requests to the replica
- `ServeResponse` is a new data model served as the output of `proxy_request()`
  method. It has required the string `status_code` used for metrics tracking. It also
  has optional `response` and `streaming_response` allowing gRPC to pass back
  response to the client

#### Benchmark
The current serve benchmark `python/ray/serve/benchmarks/microbenchmark.py` only
test http requests. In order to test gRPC requests, we need to add a new option and
change the deployment and client to use gRPC.


#### Docs
We will add new docs and code examples on how to use gRPC proxy.

## Compatibility, Deprecation, and Migration Plan
Everything should be backward compatible. All existing HTTP code path are all remain
the same. The existing gRPC ingress will be kept, but we can decide on deprecation in
favor of the new gRPC proxy. No migration plan is needed for this feature.


## Test Plan and Acceptance Criteria
- Unit test new code paths and ensure exist code path are not broken
- Manual test different Serve scenarios
  - Serve with unary response
  - Serve with streaming response
  - Serve with model composition
  - Serve with model multiplexing
- Benchmarks on performance against existing HTTP proxy
- Documentation and example usages

## (Optional) Follow-on Work
### GRPCConfigs
We will add a new configuration gRPC port used to flag to start a gRPC proxy
on a specific port (or if not starting at all). We will add a new class `GRPCOptions`
works similar to the existing `HTTPOptions` and used in `serve.start` and any
related code.

### Ray Serve Dashboard
In the current design, HTTP and gRPC requests track metrics separately. The current
dashboard only shows HTTP metrics. We will need to add a new tab to show gRPC metrics.
Or we can combine the two metrics together if we think it makes little sense to track
them separately.

### Rename HTTPState and related code
Right now we have `HTTPState` and `HTTPProxyActor` that are used for both HTTP and
gRPC proxies. We will need to rename them to be more generic and not tied to HTTP.
