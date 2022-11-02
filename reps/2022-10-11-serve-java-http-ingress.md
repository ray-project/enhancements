## Summary
### General Motivation
We want to call various methods of java deployment through the HTTP proxy, like the FastAPI ingress in python serve.
### Should this change be within `ray` or outside?
main `ray` project. Changes are made to the Ray Serve module.

## Stewardship
### Required Reviewers
@simon-mo
### Shepherd of the Proposal (should be a senior committer)
@simon-mo

## Design and Architecture
### Use case
1. Define the model with JAX-RS
```java
@Path("user")
public class UserRestService {
    @GET
    @Path("helloWorld")
    @Consumes({"application/json"})
    @Produces({"application/json"})
    public String helloWorld(String name) {
        return "hello world, " + name;
    }

    @GET
    @Path("paramPathTest/{name}")
    @Consumes({"application/json"})
    @Produces({"application/json"})
    public String paramPathTest(@PathParam("name")String name) {
        return "paramPathTest, " + name;
    }
}
```
2. Create deployment
```java
Deployment deployment =
        Serve.deployment()
            .setName("deploymentName")
            .setDeploymentDef(UserRestService.class.getName())
            .ingress("jax-rs")
            .setNumReplicas(1)
            .create();
deployment.deploy(true);
```
3. Calling deployments via HTTP
The URI is determined by the deployment name and JAX-RS @PATH annotations.
```java
curl http://127.0.0.1:8000/deploymentName/user/helloWorld?name=test
curl http://127.0.0.1:8000/deploymentName/user/paramPathTest/test
```
### HTTP ingress
#### Ingress API annotation: JAX-RS
In java application development, restful API is generally implemented through two sets of annotations, spring web or JAX-RS.

JAX-RS is a specification for implementing REST web services in Java, currently defined by the JSR-370. JAX-RS is just an API specification and has been implemented by many components: jersey,resteasy,etc.

Spring-web contains a lot of features that we don't need to use: IOC, DI, etc. The parsing of spring-web annotations depends on the spring framework.

In order not to import too many unnecessary dependencies to the user's `Callable`. We choose JAX-RS to support HTTP ingress.

For more information about JAX-RS and spring web, please click the following links.

JAX-RS: https://projects.eclipse.org/projects/ee4j.rest

Spring-web: https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-requestmapping

#### Annotation parser: Jersey
Jersey RESTful Web Services 2.x framework is open source, production quality, a framework for developing RESTful Web Services in Java that provides support for JAX-RS APIs and serves as a JAX-RS (JSR 311 & JSR 339 & JSR 370) Reference Implementation.

Jersey framework is more than the JAX-RS Reference Implementation. Jersey provides its own API that extends the JAX-RS toolkit with additional features and utilities to further simplify RESTful service and client development. Jersey also exposes numerous extension SPIs so that developers may extend Jersey to best suit their needs.

The most important thing is that the Jersey does not depend on a servlet container to run. And many other open-source frameworks need to integrate servlet containers.

For more information about Jersey, please click this link: https://eclipse-ee4j.github.io/jersey/

##### Maven dependency
```xml
<dependency>
   <groupId>org.glassfish.jersey.core</groupId>
   <artifactId>jersey-server</artifactId>
   <version>2.30.1</version>
</dependency>
<dependency>
   <groupId>org.glassfish.jersey.inject</groupId>
   <artifactId>jersey-hk2</artifactId>
   <version>2.30.1</version>
</dependency>
```
##### Jersey callable
- convert `RequestWrapper` to jersey `ContainerRequest`
- call `ApplicationHandler.apply` with `ContainerRequest`
- return `ContainerResponse.getEntity` to the HTTP proxy
```java
public class JaxrsIngressCallable {
  private ApplicationHandler app;

  public JaxrsIngressCallable(Class clazz) {
    ResourceConfig resourceConfig = new ResourceConfig(clazz);
    this.app = new ApplicationHandler(resourceConfig);
  }

  public Object call(RequestWrapper httpProxyRequest) {
    ContainerRequest jerseyRequest = convertRequestWrap2ContainerRequest(httpProxyRequest)
    ContainerResponse response = app.apply(jerseyRequest).get();
    Object rspBody = response.getEntity();
    return rspBody;
  }
}
```
### Extension of `callable`
Normally, we use the class instance as a `Callable`. But in HTTP ingress, the `Callable` we use is wrapped with the Jersey application handler. We have two different implementations of `Callable`.

In python, when we add the `@ingress` annotation to an object, a new object will be generated, that is, a new `Callable` instance will be generated.

In java, we add annotations to the class, but it can not enhance the features of `Callable`. So we need a mechanism to generate different `Callable` instances according to different ingress types. 

Here we use java SPI to implement this feature. We add a `ServeCallableProvider` SPI. 
```java
public interface ServeCallableProvider {
  /**
   * Get Callable type
   * @return Callable type
   */
  String type();

  /**
   * Generate a Callable instance
   * @param deploymentWrapper deployment info and config
   * @return Callable instance
   */
  Object buildCallable(DeploymentWrapper deploymentWrapper);

  /**
   * Get the signature of callable
   * @param deploymentWrapper  deployment info and config
   * @param callable Callable instance
   * @return
   */
  Map<String, Pair<Method, Object>> getSignatures(DeploymentWrapper deploymentWrapper, Object callable);
}
```
The current version of `Callable` is implemented as the default interface implementation. An additional implementation of the `ServeCallableProvider` is added for each additional ingress type.

### Method signature cache
Typically, our `Callable` are user-provided class instances. In JAX-RS, the `Callable` is the jersey application handler when the replica is called using an HTTP proxy. When using the serve handle to call the replica, we cannot call any methods in the class. We are not compatible with the serve handle.

On the other hand, every time we call replica, we need to use reflection to get the method that needs to be executed. This will reduce the performance and throughput of the replica

In order to solve the above problems, We need to hold the cache of method signatures.

![image](https://user-images.githubusercontent.com/11265783/195356752-95cf595b-f235-477c-b041-47b3760ad0f5.png)

Ray core uses the signature to decide which method to call. we want to be consistent with it.

When init java replica, we will parse signatures from the `Callable` class. Generate the following map. the key is the method signature, value is the pair of the method instance and the `Callable` instance.
```java
Map<String, Pair<Method, Object>> signatures;
```
If the user configures JAX-rs ingress, we will add one data to the signature cache. The key is always set to `__call__`, the left of the pair in the value is a `Method` instance of the `JaxrsIngressCallable.call` and the right of the pair is an instance of `JaxrsIngressCallable`.

For requests from HTTP proxy, the method signature will be fixed to `__call__`. The request from the serve handle will set the method signature on the client side and hit the signature cache on the server side.

## Compatibility, Deprecation, and Migration Plan
New features are incremental and do not affect any existing features. And we use SPI to make the modification of the java HTTP ingress meet the open-closed principle.
## Test Plan and Acceptance Criteria
- Unit and integration test for core components
- Benchmarks on java HTTP ingress performance

## (Optional) Follow-on Work
- `callable` support SPI
- method signature cache
- http ingress with jersey application handler
- benchmark test
