## Summary
### General Motivation
Compared to the latest Serve API in Python, Ray Serve Java lacks a major version update. Currently, the main form of the Serve API is the DAG API, which involves networking deployments through binding and deploying the graph. For example:
```python
import ray
from ray import serve
from ray.serve.dag import InputNode
from ray.serve.drivers import DAGDriver


@serve.deployment
def preprocess(inp: int) -> int:
    return inp + 1


@serve.deployment
class Model:
    def __init__(self, increment: int):
        self.increment = increment

    def predict(self, inp: int) -> int:
        return inp + self.increment


with InputNode() as inp:
    model = Model.bind(increment=2)
    output = model.predict.bind(preprocess.bind(inp))
    serve_dag = DAGDriver.bind(output)

handle = serve.run(serve_dag)
assert ray.get(handle.predict.remote(1)) == 4

```
Looking back at Java, we hope that Java can keep up with the features of the Serve API, allowing Java developers to deploy Java projects as Serve deployments and compose multiple deployments to accomplish complex online computations.
### Should this change be within `ray` or outside?
Main `ray` project. A part of java/serve.
## Stewardship

### Required Reviewers
@sihanwang41

### Shepherd of the Proposal (should be a senior committer)
@sihanwang41

## Design and Architecture

### Update Java user API to be consistent with Python
A standard Java deployment demo is shown below:
```java
// Demo 1
import io.ray.serve.api.Serve;
import io.ray.serve.deployment.Application;
import io.ray.serve.handle.RayServeHandle;

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
        Serve.deployment().setDeploymentDef(DeploymentDemo.class.getName()).create().bind();
    RayServeHandle handle = Serve.run(deployment);
    System.out.println(handle.remote().get());
  }
}

```
In this demo, a DAG node is defined through the `bind` method of the Deployment, and it is deployed using the `Serve.run` API.
Furthermore, a Deployment can bind other Deployments, and users can use the Deployment input parameters in a similar way to `RayServeHandle`. For example:
```java
// Demo 2
import io.ray.api.ObjectRef;
import io.ray.serve.api.Serve;
import io.ray.serve.deployment.Application;
import io.ray.serve.handle.RayServeHandle;

public class Driver {
  private RayServeHandle modelAHandle;
  private RayServeHandle modelBHandle;

  public Driver(RayServeHandle modelAHandle, RayServeHandle modelBHandle) {
    this.modelAHandle = modelAHandle;
    this.modelBHandle = modelBHandle;
  }

  public String call(String request) {
    ObjectRef<Object> refA = modelAHandle.remote(request);
    ObjectRef<Object> refB = modelBHandle.remote(request);
    return (String) refA.get() + refB.get();
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
    Application modelA =
        Serve.deployment().setDeploymentDef(ModelA.class.getName()).create().bind();
    Application modelB =
        Serve.deployment().setDeploymentDef(ModelB.class.getName()).create().bind();

    Application driver =
        Serve.deployment().setDeploymentDef(Driver.class.getName()).create().bind(modelA, modelB);
    Serve.run(driver);
  }
}

```
In this example, modelA and modelB are defined as two Deployments, and driver is instantiated with the corresponding `RayServeHandle` of these two Deployments. When `call` is executed, both models are invoked. Additionally, more complex graphs can be composed, for example:
```python
def preprocess(inp: int) -> int:
    return inp + 1

```

```java
// Demo 3
import io.ray.serve.api.Serve;
import io.ray.serve.deployment.Application;
import io.ray.serve.deployment.DAGDriver;
import io.ray.serve.deployment.InputNode;
import io.ray.serve.generated.DeploymentLanguage;
import io.ray.serve.handle.RayServeHandle;
import java.util.concurrent.atomic.AtomicInteger;
import org.testng.Assert;

public class Model {
  private AtomicInteger increment;

  public Model(int increment) {
    this.increment = new AtomicInteger(increment);
  }

  public int predict(int inp) {
    return inp + increment.get();
  }

  public static void main(String[] args) throws Exception {
    try (InputNode inp = new InputNode()) {
      Application model =
          Serve.deployment()
              .setDeploymentDef(Model.class.getName())
              .create()
              .bind(2);
      Application pyPreprocess =
          Serve.deployment()
              .setDeploymentDef("deployment_graph.preprocess")
              .setLanguage(DeploymentLanguage.PYTHON)
              .create()
              .bind(inp);
      Application output = model.method("predict").bind(pyPreprocess);
      Application serveDag = DAGDriver.bind(output);

      RayServeHandle handle = Serve.run(serveDag);
      Assert.assertEquals(handle.method("predict").remote(1).get(), 4);
    }
  }
}

```
In this case, two deployments are defined:
* model: a Java Deployment where the `predict` method takes an integer input and performs addition with the initialized value.
* pyPreprocess: a Python deployment that adds one to the input parameter.

During the graph composition, the output of pyPreprocess will be used as input to the `model.predict` method. `DAGDriver` acts as the Ingress Deployment and orchestrates the entire graph. Finally, the graph is deployed using `Serve.run`.

One more thing to note is the usage of `InputNode`. In Python, `InputNode` is very flexible and can represent a list, a dict, or structured object. However, in Java, it is difficult to simulate the invocation of arbitrary objects using `InputNode`, so we have made some compromises. We can simulate the invocation of a List or a Map using the `InputNode.get` method. As for structured objects, the only option is to pass the entire `InputNode` as a parameter. Here's an example:
```java
// Demo 4
import io.ray.serve.api.Serve;
import io.ray.serve.deployment.Application;
import io.ray.serve.deployment.InputNode;
import io.ray.serve.generated.DeploymentLanguage;

public class Model {

  private int weight;

  public Model() {}

  public Model(int weight) {
    this.weight = weight;
  }

  public int forward(int input) {
    return weight + input;
  }

  public static void main(String[] args) throws Exception {
    Application m1 = Serve.deployment().setDeploymentDef(Model.class.getName()).create().bind(1);
    Application m2 = Serve.deployment().setDeploymentDef(Model.class.getName()).create().bind(2);

    try (InputNode userInput = InputNode.create()) {
      Application m1Output = m1.method("forward").bind(userInput.get(0));
      Application m2Output = m2.method("forward").bind(userInput.get(1));
      Application combineOutput =
          Serve.deployment()
              .setDeploymentDef("deployment_graph.combine")
              .setLanguage(DeploymentLanguage.PYTHON)
              .create()
              .bind(m1Output, m2Output, userInput.get(2));
      
      Application graph = DAGDriver.bind(combineOutput);
      RayServeHandle handle = Serve.run(graph);
    }
  }
}

```
### Cross-language transmission of DAGDriver
In the above examples, both Demo 1 and Demo 2 are deployments of regular Java applications, which are relatively easy to implement. However, for Demo 3 and Demo 4, they involve Python `DAGDriver`, where the input of `DAGDriver` may contain `RayServeDAGHandle` that carries information for graph execution. In order to fully support graph execution orchestration through Python `DAGDriver`, it would require support for cross-language transmission of several internal core types, such as `RayServeDAGHandle`, `DeploymentMethodExecutorNode`, `DeploymentFunctionExecutorNode`, `InputAttributeNode`, and `InputNode`. This could be a significant change and needs further evaluation.

### Deploying deployments of other languages through Python API
In another REP ([Add Cpp Deployment in Ray Serve](https://github.com/ray-project/enhancements/pull/34)), it is mentioned how to deploy C++ deployments through Python. Deploying Java deployments through Python is similar. Since Java and C++ do not have the decorator mechanism like Python, a straightforward way is to directly instantiate the corresponding Deployment object:

```python
deployment_config = DeploymentConfig()
deployment_config.deployment_language = JAVA # or CPP

deployment = Deployment(_func_or_class='io.ray.serve.ExampleDeployment', name='my_deployment', config=config)
```
Alternatively, you can directly use the deployment API (with the addition of a new `language` parameter):
```python
deployment = serve.deployment(_func_or_class='io.ray.serve.ExampleDeployment', name='my_deployment', language=JAVA)

```
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
- Modify the Ray Serve Java API to support the usage of DAGs.
- Optimize the code by removing unused components and improving cross-language parameter handling.
- Support the usage of streaming.