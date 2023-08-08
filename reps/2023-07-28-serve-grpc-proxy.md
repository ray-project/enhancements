
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
@edoakes, @ericl, @pcmoritz

### Shepherd of the Proposal (should be a senior committer)

@edoakes

## Design and Architecture
### High Level Description
- We will serve gRPC from the same actors that currently serve HTTP
- User will create their own protobufs, service, and method
- User will start Serve's gRPC proxy with port and their protobuf servicer functions
- User will call gRPC services using user defined protobufs, and with optional metadata
- Client will receive user defined protobuf as response
- Serve will also return request id back to the client with trailing metadata
- gRPC deployments will have full feature parity with HTTP, including
  streaming responses, model multiplexing, and model composition support


### Example code
#### Serve Config gRPC Option
```python
class gRPCOptionsSchema(BaseModel, extra=Extra.forbid):
  """Options to start the gRPC Proxy with."""

  port: int = Field(
    default=-1,
    description=(
      "Port for gRPC server. Will only start gRPC server if set. Cannot be "
      "updated once Serve has started running. Serve must be shut down and "
      "restarted with the new port instead."
    ),
  )
  grpc_servicer_functions: List[str] = Field(
    default=[],
    description=(
      "The servicer functions used to add the method handlers to the gRPC server."
      "Default to empty list, which means no gRPC methods will be added"
      "and no gRPC server will be started. The servicer functions need to be"
      "importable from the context of where Serve is running."
    ),
  )

```

#### Accepted Metadata
- `application`: The name of the application to route to. If not passed, first
  application will be used automatically.
- `request_id`: The request id to track the request. Feature parity with http proxy.
- `multiplexed_model_id`: The model id to do model multiplexing. Feature parity with
  http proxy.

#### Unary gRPC example
```proto
// user_defined_protos.proto

syntax = "proto3";

option java_multiple_files = true;
option java_package = "io.ray.examples.user_defined_protos";
option java_outer_classname = "UserDefinedProtos";

package userdefinedprotos;

message UserDefinedMessage {
  string name = 1;
  string foo = 2;
  int64 num = 3;
}

message UserDefinedResponse {
  string greeting = 1;
  int64 num_x2 = 2;
}

message UserDefinedMessage2 {}

message UserDefinedResponse2 {
  string greeting = 1;
}

service UserDefinedService {
  rpc __call__(UserDefinedMessage) returns (UserDefinedResponse);
  rpc Method1(UserDefinedMessage) returns (UserDefinedResponse);
  rpc Method2(UserDefinedMessage2) returns (UserDefinedResponse2);
  rpc Streaming(UserDefinedMessage) returns (stream UserDefinedResponse);
}

```

```yaml
# config.yaml

grpc_options:

  port: 9002

  grpc_servicer_functions:

    - user_defined_protos_pb2_grpc.add_UserDefinedServiceServicer_to_server

applications:

  - name: app1

    route_prefix: /

    import_path: test_deployment:g

    runtime_env: {}

```

```python
# test_deployment.py

# Users need to include their custom message type which will be embedded in the request.
from user_defined_protos_pb2 import (
  UserDefinedMessage,
  UserDefinedMessage2,
  UserDefinedResponse,
  UserDefinedResponse2,
)

from ray import serve


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

  def method1(self, user_message: UserDefinedMessage) -> UserDefinedResponse:
    greeting = f"Hello {user_message.foo} from method1"
    num_x2 = user_message.num * 3
    user_response = UserDefinedResponse(
      greeting=greeting,
      num_x2=num_x2,
    )
    return user_response

  def method2(self, user_message: UserDefinedMessage2) -> UserDefinedResponse2:
    greeting = "This is from method2"
    user_response = UserDefinedResponse(greeting=greeting)
    return user_response


g = GrpcDeployment.options(name="grpc-deployment").bind()

```

```python
# test_client.py

import grpc

# Users define their custom protobuf and services
from user_defined_protos_pb2 import UserDefinedMessage, UserDefinedMessage2
from user_defined_protos_pb2_grpc import UserDefinedServiceStub

# The gRPC port is defined in the config file
channel = grpc.insecure_channel("localhost:9002")

# The service stub is generated by user's proto file
stub = UserDefinedServiceStub(channel)

test_in = UserDefinedMessage(
  name="genesu",
  num=88,
  foo="bar",
)
metadata = (
  # ("application", "app1_grpc-deployment"),  # Optional, default to first deployment
  # ("request_id", "123"),  # Optional, feature parity w/ http proxy
  # ("multiplexed_model_id", "456"),  # Optional, feature parity w/ http proxy
)
# __call__ method is defined by user's proto file and
# match with deployment method `__call__()`
response, call = stub.__call__.with_call(request=test_in, metadata=metadata)
print(call.trailing_metadata())  # Request id is returned in the trailing metadata
print("Output type:", type(response))  # Response is a type of UserDefinedResponse
print("Full output:", response)
print("Output greeting field:", response.greeting)
print("Output num_x2 field:", response.num_x2)

# Method1 method is defined by user's proto file and
# match with deployment method `method1()`
response, call = stub.Method1.with_call(request=test_in, metadata=metadata)

print(call.trailing_metadata())  # Request id is returned in the trailing metadata
print("Output type:", type(response))  # Response is a type of UserDefinedResponse
print("Full output:", response)
print("Output greeting field:", response.greeting)
print("Output num_x2 field:", response.num_x2)

test_in = UserDefinedMessage2()
# Method2 method is defined by user's proto file and
# match with deployment method `method2()`
response, call = stub.Method2.with_call(request=test_in, metadata=metadata)

print(call.trailing_metadata())  # Request id is returned in the trailing metadata
print("Output type:", type(response))  # Response is a type of UserDefinedResponse2
print("Full output:", response)
print("Output greeting field:", response.greeting)

```

#### Streaming gRPC example
```python
# test_deployment.py

import time
from typing import Generator

# Users need to include their custom message type which will be embedded in the request.
from user_defined_protos_pb2 import UserDefinedMessage, UserDefinedResponse

from ray import serve


@serve.deployment
class GrpcDeployment:
  def streaming(
          self, user_message: UserDefinedMessage
  ) -> Generator[UserDefinedResponse, None, None]:
    for i in range(10):
      greeting = f"{i}: Hello {user_message.name} from {user_message.foo}"
      num_x2 = user_message.num * 2 + i
      user_response = UserDefinedResponse(
        greeting=greeting,
        num_x2=num_x2,
      )
      yield user_response

      time.sleep(0.1)


g = GrpcDeployment.options(name="grpc-deployment").bind()

```

```python
# test_client.py

import grpc

# Users define their custom protobuf and services
from user_defined_protos_pb2 import UserDefinedMessage
from user_defined_protos_pb2_grpc import UserDefinedServiceStub

# The gRPC port is defined in the config file
channel = grpc.insecure_channel("localhost:9002")

# The service stub is generated by user's proto file
stub = UserDefinedServiceStub(channel)

test_in = UserDefinedMessage(
  name="genesu",
  num=88,
  foo="bar",
)
metadata = (
  # ("application", "app1_grpc-deployment"),  # Optional, default to first deployment
  # ("request_id", "123"),  # Optional, feature parity w/ http proxy
  # ("multiplexed_model_id", "456"),  # Optional, feature parity w/ http proxy
)

# Streaming method is defined by user's proto file and
# match with deployment method `streaming()`
responses = stub.Streaming(test_in, metadata=metadata)

for response in responses:
  print("Output type:", type(response))  # Response is a type of UserDefinedResponse
  print("Full output:", response)
  print("Output greeting field:", response.greeting)
  print("Output num_x2 field:", response.num_x2)
print(responses.trailing_metadata())  # Request id is returned in the trailing metadata


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
- `gRPCProxy` is a subclass from `GenericProxy` includes specific implementations of
  - gRPC entrypoints one for unary and another for streaming. The only difference
    between these two methods are the return type being a unary object or stream object.
    These two methods also wrap the gRPC input objects into a `ServeRequest` object
    to use in shared `proxy_request()` method on the `GenericProxy`
  - Setup request context and handle based on the metadata
  - Methods to process request and return the user defined protobuf object
- `RayServeHandle`, `Router`, and `Scheduler` all kept unchanged except
   added a new flag `serve_grpc_request` on the `HandleOptions` and `RequestMetadata`
   to signal to the replicas when it’s a gRPC request
- `Replica` handling HTTP requests all continue to use existing methods
  - New methods are added to handle requests in gRPC protocol
  - New if-statement is added to use gRPC methods when the `serve_grpc_request`
    flag is set true

![serve_data_flow_chart](https://docs.google.com/drawings/d/e/2PACX-1vTtg5xhPRvzghhNuOsFCPCllrWWZwVSqEGVpL3xd3ggTiFSKquW4x0sEmEjT3hsDvijmEo0ZOTiTVJO/pub?w=1527&h=866)
- HTTP clients continue to send requests through `HTTPProxy` using ASGI protocol.
  `HTTPProxy` will return the response in the ASGI `Send` object
- gRPC clients send user defined protobuf to the `gRPCProxy` to send request
  to the replicas. `gRPCProxy` will return back user defined protobuf
  - Metadata from the client to the gRPC server contains `application` for which
    application to route to, `request_id` for tracking the request, and
    `multiplexed_model_id` for doing model multiplexing. All those are optional to
    a request.
  - Trailing Metadata from the server contains `request_id` for tracking the request
- The rest are existing code besides added a new `gRPCRequest` object to be used
  in place of `StreamingHTTPRequest` object in `RayServeHandle` and `Router`

### Change Details
You can find the prototype PR here: [ray-project/ray#37310](https://github.com/ray-project/ray/pull/37310)

#### Proxy
- `GenericProxy` is refactored from `HTTPProxy`. It includes only the common variables
  and methods used by both proxies. It served as the parent class for both `HTTPProxy`
  and `gRPCProxy` to inherit from.
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
- `gRPCProxy` is a subclass from `GenericProxy`.
  - `unary_unary()` is the entrypoint for unary gRPC requests. It takes user defined
    protobuf as input and returns user defined protobuf as the response. It wraps
    the input and the metadata into a `ServeRequest` object and calls `proxy_request()`
  - `unary_stream()` is the entrypoint for streaming gRPC requests. It takes
    user defined protobuf as the input and returns a generator of user defined
    protobuf. It wraps the input into a `ServeRequest` object and calls
    `proxy_request()`
  - `setup_request_context_and_handle` is based on the metadata.
  - `send_request_to_replica_streaming()` will handle request and return the
    response as `ServeResponse` object.
- `HTTPProxyActor` will be changed to start both `HTTPProxy` and `gRPCProxy`.
  `gRPCProxy` is only setup when the `gRPCOptions` is provided in the config.
- `LongestPrefixRouter` to add a new method `match_target()` to help gRPC look up for
  the routes from the application name

#### Handle and Router
New flag `serve_grpc_request` is added to the `HandleOptions` and to the
`RequestMetadata` to signal to the replicas when it’s a gRPC request. This flag
is boolean defaulting to false. It's only set when `gRPCProxy` is setting up the
handle.

#### Replica
- `call_user_method_grpc_unary()` is added to handle unary gRPC requests
- `call_user_method_with_grpc_streaming()` is added to handle streaming gRPC requests
- A new if-statement is added to use gRPC methods when the `serve_grpc_request`
  flag is set true in `handle_request()` and in `handle_request_streaming()`

#### Data Models
- `gRPCRequest` is added to replace `StreamingHTTPRequest` in `RayServeHandle`
  and `Router` to be used for gRPC requests. It contains a `grpc_user_request`
  field to store the byte input for the replica and a `grpc_proxy_handle` field
  to store the ray serve handle to be used for gRPC requests.
- `ServeRequest` is a new data model served as the input to `proxy_request()`
  method so any types of proxy can share the same GA entry point.
- `ASGIServeRequest` is a subclass of `ServeRequest` used by `HTTPProxy`.
  It contains the implementations to extract attributes from ASGI input objects for
  sending requests to the replica
- `gRPCServeRequest` is a subclass of `ServeRequest` used by `gRPCProxy`.
  It contains the implementations to extract attributes from gRPC input objects
  for sending requests to the replica
- `ServeResponse` is a new data model served as the output of `proxy_request()`
  method. It has required the string `status_code` used for metrics tracking. It also
  has optional `response` and `streaming_response` allowing gRPC to pass response
  back to the client

### gRPCOptions
This is a new config option we added to the serve config. It contains two fields:
- `port`: TPort for gRPC server. Will only start gRPC server if set. Cannot be
  updated once Serve has started running. Serve must be shut down and restarted
  with the new port instead.
- `grpc_servicer_functions`: The servicer functions used to add the method handlers
  to the gRPC server. Default to empty list, which means no gRPC methods will be
  added and no gRPC server will be started. The servicer functions need to be
  importable from the context of where Serve is running.

If Serve is started with `gRPCOptions` provided, the `HTTPProxyActor` will start
the gRPC proxy with the provided port and the provided servicer functions.

#### gRPC Util
This is where our implementation of gRPC the serialization and deserialization lives.
We created our custom function `create_serve_grpc_server()` and custom class
`gRPCServer` to serialize and deserialize user defined protobuf.
- `create_serve_grpc_server()` is used as a factory function to create custom
  `gRPCServer` instance. The function will take a method handler factory and
  the factory will create a callable `unary_unary()` for unary request and
  `unary_stream()` for streaming request.
- `gRPCServer` is subclass from `grpc.Server` and it overrides
  `add_generic_rpc_handlers()` to use our method handlers and to disable
  `response_serializer` so gRPC proxy can return raw user defined protobuf bytes to
  the user.

#### Benchmark
We will benchmark the performance of the gRPC proxy against the existing HTTP proxy
with new scripts.

#### Docs
We will add new docs and code examples on how to use gRPC proxy.

#### Dependencies
We previously implemented the direct ingress with `gRPCIngress` using Ray's default
gRPC library. We will need to include gRPC library to Serve's default libraries
to ensure this new feature continue to work after Ray drop it's gRPC library.

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
### Ray Serve Dashboard
In the current design, HTTP and gRPC requests track metrics separately. The current
dashboard only shows HTTP metrics. We will need to add a new tab to show gRPC metrics.
Or we can combine the two metrics together if we think it makes little sense to track
them separately.

### Rename HTTPState and related code
Right now we have `HTTPState` and `HTTPProxyActor` that are used for both HTTP and
gRPC proxies. We will need to rename them to be more generic and not tied to HTTP.
