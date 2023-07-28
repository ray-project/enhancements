
## Summary - Serve gRPC Proxy
Improve Ray Serve GA on gRPC by offering a gRPC proxy to the users. The current
experimental gRPC ingress is a direct ingress to the replica. It lacks features we
built into HTTP proxy to keep Serve robust and generally available. This REP
describes how we can provide more performant gRPC service while reusing the GA
features we built with HTTP proxy.

### General Motivation
GRPC is generally a more performant way to talk to services. It utilizes the 
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
will be able to start a gRPC service without changes from outside. To offer 
Anyscale users the same ability to serve with gRPC, we would also need to 
follow up on adding new ALB entry and listeners to the gRPC port.


## Stewardship
### Required Reviewers
@edoakes, @ericl(?), @pcmoritz(?)

### Shepherd of the Proposal (should be a senior committer)

@edoakes

## Design and Architecture

### Example code 
#### Service using protobuf serialization
```python
# test_deployment.py
from ray import serve
from ray.serve.generated import serve_pb2


@serve.deployment
class GrpcDeployment:
    def __call__(self, my_input: bytes) -> bytes:
        request = serve_pb2.TestIn()
        request.ParseFromString(my_input)
        greeting = f"Hello {request.name} from {request.foo}"
        num_x2 = request.num * 2
        output = serve_pb2.TestOut(
            greeting=greeting,
            num_x2=num_x2,
        )
        return output.SerializeToString()


g = GrpcDeployment.options(name="grpc-deployment").bind()


# test_client.py
import time

import grpc

from ray.serve.generated import serve_pb2, serve_pb2_grpc

start_time = time.time()
channel = grpc.insecure_channel("localhost:9000")
stub = serve_pb2_grpc.RayServeServiceStub(channel)

test_in = serve_pb2.TestIn(
    name="genesu",
    num=88,
    foo="bar",
)
response = stub.Predict(
    serve_pb2.RayServeRequest(
        application="default_grpc-deployment",
        user_request=test_in.SerializeToString(),
        # request_id="123",
        # multiplexed_model_id="123",
    )
)
print("Time taken:", time.time() - start_time)
print("Output type:", type(response.user_response))
print("Full output:", response.user_response)
print("request_id:", response.request_id)

test_out = serve_pb2.TestOut()
test_out.ParseFromString(response.user_response)
print("Output greeting field:", test_out.greeting)
print("Output num_x2 field:", test_out.num_x2)
```

#### Service using pickle serialization
```python
# test_deployment.py
import pickle

from ray import serve


@serve.deployment
class GrpcDeploymentNoProto:
  def __call__(self, my_input: bytes) -> bytes:
    request = pickle.loads(my_input)
    greeting = f"Hello {request['name']} from {request['foo']}"
    num_x2 = request["num"] * 2
    output = {
      "greeting": greeting,
      "num_x2": num_x2,
    }
    return pickle.dumps(output)


g2 = GrpcDeploymentNoProto.options(name="grpc-deployment-no-proto").bind()


# test_client.py
import pickle
import time

import grpc

from ray.serve.generated import serve_pb2, serve_pb2_grpc

start_time = time.time()
channel = grpc.insecure_channel("localhost:9000")
stub = serve_pb2_grpc.RayServeServiceStub(channel)

test_in = {
  "name": "genesu",
  "num": 88,
  "foo": "bar",
}
response = stub.Predict(
  serve_pb2.RayServeRequest(
    application="default_grpc-deployment-no-proto",
    user_request=pickle.dumps(test_in),
    request_id="123",
    # multiplexed_model_id="123",
  )
)
print("Time taken:", time.time() - start_time)
print("Output type:", type(response.user_response))
print("Full output:", response.user_response)
print("request_id:", response.request_id)

test_out = pickle.loads(response.user_response)
print("Output greeting field:", test_out["greeting"])
print("Output num_x2 field:", test_out["num_x2"])

```

#### Service streaming response
```python
# test_deployment.py
import time
from typing import Generator

from ray import serve
from ray.serve.generated import serve_pb2


@serve.deployment
class GrpcDeploymentStreamingResponse:
  def __call__(self, my_input: bytes) -> Generator[bytes, None, None]:
    print("my_input", my_input)
    request = serve_pb2.TestIn()
    request.ParseFromString(my_input)
    for i in range(10):
      greeting = f"{i}: Hello {request.name} from {request.foo}"
      num_x2 = request.num * 2 + i
      output = serve_pb2.TestOut(
        greeting=greeting,
        num_x2=num_x2,
      )
      yield output.SerializeToString()

      time.sleep(0.1)


g3 = GrpcDeploymentStreamingResponse.options(
  name="grpc-deployment-streaming-response"
).bind()


# test_client.py
import time

import grpc

from ray.serve.generated import serve_pb2, serve_pb2_grpc

start_time = time.time()
channel = grpc.insecure_channel("localhost:9000")
stub = serve_pb2_grpc.RayServeServiceStub(channel)

test_in = serve_pb2.TestIn(
  name="genesu",
  num=88,
  foo="bar",
)
responses = stub.PredictStreaming(
  serve_pb2.RayServeRequest(
    application="default_grpc-deployment-streaming-response",
    user_request=test_in.SerializeToString(),
    # request_id="123",
    # multiplexed_model_id="123",
  )
)
print("Time taken:", time.time() - start_time)
for response in responses:
  print("Output type:", type(response.user_response))
  print("Full output:", response.user_response)
  print("request_id:", response.request_id)

  test_out = serve_pb2.TestOut()
  test_out.ParseFromString(response.user_response)
  print("Output greeting field:", test_out.greeting)
  print("Output num_x2 field:", test_out.num_x2)

```

#### Service model composition 
```python
# test_deployment.py
import struct
from typing import Dict

import ray
from ray import serve
from ray.serve.handle import RayServeDeploymentHandle


@serve.deployment
class FruitMarket:
  def __init__(
          self,
          _orange_stand: RayServeDeploymentHandle,
          _apple_stand: RayServeDeploymentHandle,
  ):
    self.directory = {
      "ORANGE": _orange_stand,
      "APPLE": _apple_stand,
    }

  async def __call__(self, inputs: bytes) -> bytes:
    fruit_amounts = pickle.loads(inputs)
    costs = await self.check_price(fruit_amounts)
    return struct.pack("f", costs)

  async def check_price(self, inputs: Dict[str, int]) -> float:
    costs = 0
    for fruit, amount in inputs.items():
      if fruit not in self.directory:
        return
      fruit_stand = self.directory[fruit]
      ref: ray.ObjectRef = await fruit_stand.remote(int(amount))
      result = await ref
      costs += result
    return costs


@serve.deployment
class OrangeStand:
  def __init__(self):
    self.price = 2.0

  def __call__(self, num_oranges: int):
    return num_oranges * self.price


@serve.deployment
class AppleStand:
  def __init__(self):
    self.price = 3.0

  def __call__(self, num_oranges: int):
    return num_oranges * self.price


orange_stand = OrangeStand.bind()
apple_stand = AppleStand.bind()
g4 = FruitMarket.options(name="grpc-deployment-multi-app").bind(
  orange_stand, apple_stand
)


# test_client.py
import pickle
import struct
import time

import grpc

from ray.serve.generated import serve_pb2, serve_pb2_grpc

start_time = time.time()
channel = grpc.insecure_channel("localhost:9000")
stub = serve_pb2_grpc.RayServeServiceStub(channel)

input = {
  "ORANGE": 10,
  "APPLE": 3,
}
response = stub.Predict(
  serve_pb2.RayServeRequest(
    application="default_grpc-deployment-multi-app",
    user_request=pickle.dumps(input),
    request_id="123",
    # multiplexed_model_id="123",
  )
)
print("Time taken:", time.time() - start_time)
print("Output type:", type(response.user_response))
print("Full output:", response.user_response)
print("request_id:", response.request_id)

response = struct.unpack("f", response.user_response)
print("Output:", response)

```

#### Service model multiplexing
```python
# test_deployment.py
from ray import serve


@serve.deployment
class GrpcDeploymentMultiplexing:
  @serve.multiplexed(max_num_models_per_replica=3)
  async def get_model(self, model_id: str) -> str:
    return f"loading model: {model_id}"

  async def __call__(self, request: bytes) -> bytes:
    model_id = serve.get_multiplexed_model_id()
    model = await self.get_model(model_id)
    return model.encode("utf-8")


g5 = GrpcDeploymentMultiplexing.options(name="grpc-deployment-multiplexing").bind()


# test_client.py
import time

import grpc

from ray.serve.generated import serve_pb2, serve_pb2_grpc

start_time = time.time()
channel = grpc.insecure_channel("localhost:9000")
stub = serve_pb2_grpc.RayServeServiceStub(channel)

response = stub.Predict(
  serve_pb2.RayServeRequest(
    application="default_grpc-deployment-multiplexing",
    user_request=b"",
    # request_id="123",
    multiplexed_model_id="456",
  )
)
print("Time taken:", time.time() - start_time)
print("Output type:", type(response.user_response))
print("Full output:", response.user_response)
print("request_id:", response.request_id)

user_response = response.user_response.decode()
print("user_response", user_response)

```

### Design Diagram
![grpc_proxy_design](https://docs.google.com/drawings/d/e/2PACX-1vQl8UgOdg3Td-RKG208v6dlfSj03JHPnfglEi6SpsqTwOHowiA-0sCSUx3PZ5wG842aLcyTiFelqrlt/pub?w=1481&h=1047)
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
  - GRPC entrypoints one for unary and another for streaming. GRPC requires a 
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
- GRPC clients send predefined protobuf `RayServeRequest` to the `GRPCProxy`. 
  `GRPCProxy` will return back a predefined protobuf `RayServeResponse` object.
  - `RayServeRequest` contains `user_request` as byte to pass to the replicas, 
    `application` for which application to route to, `request_id` for tracking 
    the request, and `multiplexed_model_id` for doing model multiplexing
 - `RayServeResponse` contains `user_response` as byte to return to the client 
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
  setup [GRPCConfigs](#GRPCConfigs)
- `LongestPrefixRouter` to add a new method `match_target()` to help gRPC look up for
  the routes from the application name

#### Handle and Router
New flag `serve_grpc_request` is added to the `HandleOptions` and to the
`RequestMetadata` to signal to the replicas when it’s a gRPC request. This flag
is boolean defaulting to false. It's only set when `GRPCProxy` is setting up the 
handle. 

#### Replica
- `call_user_method_grpc_unary()` is added to handle unary grpc requests
- `call_user_method_with_grpc_stream()` is added to handle streaming grpc requests
- A new if-statement is added to use grpc methods when the `serve_grpc_request`
  flag is set true in `handle_request` and in `handle_request_streaming`

#### Data Models
- `GRPCRequest` is added to replace `StreamingHTTPRequest` in `RayServeHandle` 
  and `Router` to be used for gRPC requests. It contains a `grpc_user_request`
  field to store the byte input for the replica and a `grpc_proxy_handle` field
  to store the ray serve handle to be used for gRPC requests.
- `RayServeRequest` is a protobuf object to be used for gRPC requests. It contains 
  `user_request` as byte to pass to the replicas, `application` for which 
  application to route to, `request_id` for tracking the request, and 
  `multiplexed_model_id` for doing model multiplexing
- `RayServeResponse` is a protobuf object to be used for gRPC requests. It contains 
  `user_response` as byte to return to the client and `request_id` for tracking 
  the request
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
  It contains the implementations to extract attributes from GRPC input objects 
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
favor of the new gRPC proxy.


## Test Plan and Acceptance Criteria
- Unit test new code paths and ensure exist code path are not broken
- Manual test different Serve scenarios
 - Serve with protobuf
 - Serve without protobuf
 - Serve with streaming response
 - Serve with model composition
 - Serve with model multiplexing
 - Serve on Anyscale platform
- Benchmarks on performance against existing HTTP proxy
- Documentation and example usages

## (Optional) Follow-on Work
### GRPCConfigs
We will add a new configuration grpc port used to flag to start a gRPC proxy 
on a specific port (or starting at all). We will add a new class `GRPCOptions` 
works similar to existing `HTTPOptions` and used in `serve.start` and any related code.

### ALB expose grpc port on product
OSS will be able to start a gRPC proxy without this change. However, Anyscale users 
requires a new ALB entry and listeners to the gRPC port to be able to use gRPC on
Anyscale Services.

### Ray Serve Dashboard
In the current design, http and grpc requests track metrics separately. The current
dashboard only shows http metrics. We will need to add a new tab to show grpc metrics.
Or we can combine the two metrics together if we think it makes little sense to track
them separately. 

### Rename HTTPState and related code
Right now we have `HTTPState` and `HTTPProxyActor` that are used for both HTTP and 
gRPC. We will need to rename them to be more generic and not tied to HTTP.

### DAGDriver Support (?)
DAGDriver is currently wrapped with ASGI protocol. Do we need to support gRPC protocol
for DAGDriver?
