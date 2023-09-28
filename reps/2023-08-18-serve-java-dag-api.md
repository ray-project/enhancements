## Summary
### General Motivation
Compared to the latest Serve API in Python, Ray Serve Java lacks a major version update. Currently, the main form of the Serve API is the DAG API, which involves networking deployments through binding and deploying the graph. For example:
```python
import ray
from ray import serve
from ray.serve.handle import RayServeHandle, DeploymentHandle


@serve.deployment
def preprocess(inp: int) -> int:
    return inp + 1


@serve.deployment
class Model:
    def __init__(self, preprocess_handle: RayServeHandle, increment: int):
        self.preprocess_handle: DeploymentHandle = preprocess_handle.options(use_new_handle_api=True)
        self.increment = increment

    async def predict(self, inp: int) -> int:
        preprocessed = await self.preprocess_handle.remote(inp)
        return preprocessed + self.increment

app = Model.bind(preprocess.bind(), increment=2)
handle = serve.run(app)
assert ray.get(handle.predict.remote(1)) == 4

```
Looking back at Java, we hope that Java can keep up with the features of the Serve API, allowing Java developers to deploy Java projects as Serve deployments and compose multiple deployments to accomplish complex online computations.
### Should this change be within `ray` or outside?
Main `ray` project. A part of java/serve.
## Stewardship

### Required Reviewers
@sihanwang41 @edoakes

### Shepherd of the Proposal (should be a senior committer)
@sihanwang41 @edoakes

## Design and Architecture

### Update Java User API to be Consistent with Python
A standard Java deployment demo is shown below:
```java
import io.ray.serve.api.Serve;
import io.ray.serve.deployment.Application;
import io.ray.serve.handle.DeploymentHandle;

public class DeploymentDemo {
  private String msg;

  public DeploymentDemo(String msg) {
    this.msg = msg;
  }

  public String call() {
    return msg;
  }

  public static void main(String[] args) {
    Application deployment =
        Serve.deployment().setDeploymentDef(DeploymentDemo.class.getName()).bind();
    DeploymentHandle handle = Serve.run(deployment).get();
    System.out.println(handle.remote().result());
  }
}

```
In this demo, a deployment is defined through the `bind` method, and it is deployed using the `Serve.run` API.
Furthermore, a deployment can bind other deployments, and users can use the deployment input parameters in a similar way to the `DeploymentHandle`. For example:
```java
import io.ray.serve.api.Serve;
import io.ray.serve.deployment.Application;
import io.ray.serve.handle.DeploymentHandle;
import io.ray.serve.handle.DeploymentResponse;

public class Driver {
  private DeploymentHandle modelAHandle;
  private DeploymentHandle modelBHandle;

  public Driver(DeploymentHandle modelAHandle, DeploymentHandle modelBHandle) {
    this.modelAHandle = modelAHandle;
    this.modelBHandle = modelBHandle;
  }

  public String call(String request) {
    DeploymentResponse responseA = modelAHandle.remote(request);
    DeploymentResponse responseB = modelBHandle.remote(request);
    return (String) responseA.result() + responseB.result();
  }

  public static class ModelA {
    public String call(String msg) {
      return msg;
    }
  }

  public static class ModelB {
    public String call(String msg) {
      return msg;
    }
  }

  public static void main(String[] args) {
    Application modelA = Serve.deployment().setDeploymentDef(ModelA.class.getName()).bind();
    Application modelB = Serve.deployment().setDeploymentDef(ModelB.class.getName()).bind();

    Application driver =
        Serve.deployment().setDeploymentDef(Driver.class.getName()).bind(modelA, modelB);
    Serve.run(driver);
  }
}

```
In this example, the modelA and modelB are defined as two Deployments, and the driver is instantiated with the corresponding `DeploymentHandle` of these two Deployments. When `call` is executed, both models are invoked. Additionally, it is evident that `DeploymentHandle.remote` returns `DeploymentResponse` instead of `ObjectRef`. The result can be accessed through `DeploymentResponse.result`.

### Deploying Deployments of the Other Languages through Python API
In another REP ([Add Cpp Deployment in Ray Serve](https://github.com/ray-project/enhancements/pull/34)), it is mentioned how to deploy C++ deployments through Python. Deploying Java deployments through Python is similar. Since Java and C++ do not have the decorator mechanism like Python, a straightforward way is to directly use the `serve.deployment` API (with the addition of a new `language` parameter):

```python
deployment = serve.deployment('io.ray.serve.ExampleDeployment', name='my_deployment', language=JAVA)

```
### Deploying through the Config File
Using a config file, we can deploy the Serve application through the CLI. For Java deployment, we can also describe it in a Python file and generate the corresponding deployment configuration in the config file. Let's take the example of the text.py file:
```python
from ray import serve
from ray.serve.generated.serve_pb2 import JAVA


@serve.deployment
class Hello:
    def __call__(self) -> str:
        return "Hello"


world_java = serve.deployment('io.ray.serve.World', language=JAVA)


@serve.deployment
class Ingress:
    def __init__(self, hello_handle, world_handle):
        self._hello_handle = hello_handle.options(
            use_new_handle_api=True,
        )
        self._world_handle = world_handle.options(
            use_new_handle_api=True,
        )

    async def __call__(self) -> str:
        hello_response = self._hello_handle.remote()
        world_response = self._world_handle.remote()
        return (await hello_response) + (await world_response)


hello = Hello.bind()
world = world_java.bind()

app = Ingress.bind(hello, world)

```
In this code, we define the Java Deployment `World` using the Python API and then bind it with a Python Deployment `Hello` into the `Ingress` to form an app. By using the `serve build` command, the Serve config file can be generated.
```shell
$ serve build text:app -o serve_config.yaml
```
The generated config file looks like this:
```yaml
proxy_location: EveryNode

http_options:
  host: 0.0.0.0
  port: 8000

grpc_options:
  port: 9000
  grpc_servicer_functions: []

applications:
- name: app1
  route_prefix: /
  import_path: text:app
  runtime_env: {}
  deployments:
  - name: Hello
  - name: World
  - name: Ingress

```
By using this serve config file, the Application can be deployed through the "serve run" command:
```shell
$ serve run serve_config.yaml
```

### Serve Handle C++ Core

In the design of C++ Deployment, it also includes the C++ implementation of Serve Handle. After the implementation, it can be reused as the core of Serve Handle by other languages (Python and Java) to avoid maintaining duplicate logic in the three languages. For the complete design, we will continue to supplement it in the "[Cpp Deployment Design](https://github.com/ray-project/enhancements/pull/34)" or another new document.

## Compatibility, Deprecation, and Migration Plan
In Java, the old API will be marked with the @Deprecated annotation, for example:
```java
public class Deployment {
  @Deprecated
  public void deploy(boolean blocking) {
    Serve.getGlobalClient()
        .deploy(
            name,
            deploymentDef,
            initArgs,
            rayActorOptions,
            config,
            version,
            prevVersion,
            routePrefix,
            url,
            blocking);
  }
}
```
This is also done to maintain consistency with the Python API, and to allow for easy removal of the deprecated API in the future.
## Test Plan and Acceptance Criteria
Related test cases will be provided under ray/java/serve, and they will cover the three scenarios mentioned above.
## (Optional) Follow-on Work
- Optimize the code by removing unused components and improving cross-language parameter handling.
- Support the usage of streaming.
