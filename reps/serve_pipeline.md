
## Summary - Serve Pipeline
### General Motivation
Production machine learning serving pipelines are getting longer and wider. They often consist of multiple, or even tens of models collectively making a final prediction, such as image / video content classification and tagging, fraud detection pipeline with multiple policies and models, multi-stage ranking and recommendation, etc.

Meanwhile, the size of a model is also growing beyond the memory limit of a single machine due to the exponentially growing number of parameters, such as GPT-3, sparse feature embeddings in recsys models such that the ability to do disaggregated and distributed inference is desirable and future proof.

We want to leverage the programmable and general purpose distributed computing ability of Ray, double down on its unique strengths (scheduling, communication and shared memory) to facilitate authoring, orchestrating, scaling and deployment of complex serving pipelines under one set of DAG API, so a user can program & test multiple models or multiple shards of a single large model dynamically, deploy to production at scale, and upgrade individually.
#### Key requirements:
- Provide the ability to author a DAG of serve nodes to form a complex inference graph.
- Pipeline authoring experience should be fully python programmable with support for dynamic selection, control flows, user business logic, etc.
- DAG can be instantiated and locally executed using tasks and actors API
- DAG can be deployed via declarative and idempotent API, individual nodes can be reconfigured and scaled indepenently. 

### Should this change be within `ray` or outside?
main `ray` project. Changes are made to Ray Core and Ray Serve level.

## Stewardship
### Required Reviewers
The proposal will be open to the public, but please suggest a few experience Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@ericl, @edoakes, @simon-mo, @jiaodong

### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

@ericl

## Design and Architecture

### Example - Diagram 

We want to author a simple diamond-shaped DAG where user provided inputs is send to two models (m1, m2) where each access partial or idential input, and also forward part of original input to the final ensemble stage to compute final output. 

                   m1.forward(dag_input[0])
                /                          \
        dag_input ----- dag_input[2] ------ ensemble -> dag_output
                \                          /  
                   m2.forward(dag_input[1])  
                                           
         
### Example - Code

Classes or functions decorated by ray can be directly used in Ray DAG building.
```python
@ray.remote
class Model:
def __init__(self, val):
    self.val = val
def forward(self, input):
    return self.val * input

@ray.remote
def ensemble(a, b, c):
    return a + b + c

async def request_to_data_int(request: starlette.requests.Request):
    data = await request.body()
    return int(data)

# Args binding, DAG building and input preprocessor definition
with ServeInputNode(preprocessor=request_to_data_int) as dag_input:
    m1 = Model.bind(1)
    m2 = Model.bind(2)
    m1_output = m1.forward.bind(dag_input[0])
    m2_output = m2.forward.bind(dag_input[1])
    ray_dag = ensemble.bind(m1_output, m2_output, dag_input[2])
```

A DAG authored with Ray DAG API should be locally executable just by Ray Core runtime.

```python
# 1*1 + 2*2 + 3
assert ray.get(ray_dag.execute(1, 2, 3)) == 8
```

A Ray DAG can be built into an `serve application` that contains all nodes needed.
```python
# Build, configure and deploy
app = serve.pipeline.build(ray_dag)
```

Configure individual deployments in app, with same variable name used in ray_dag.
```python
app.m1.set_options(num_replicas=3)
app.m2.set_options(num_replicas=5)
```
 
We reserve the name and generate a serve `ingress` deployment that takes care of HTTP / gRPC, input schema validation, adaption, etc. It's our python interface to configure pipeline ingress. 
```python
app.ingress.set_options(num_replicas=10)
 
# Translate to group_deploy behind the scene
app_handle = app.deploy()
 
# Serve App is locally executable
assert ray.get(app_handle.remote(1, 2, 3)) == 8
```

A serve pipeline application can be built into a YAML file for structured deployment, and configurable by the Ops team by directly mutating configurable fields without deep knowledge or involvement of model code in the pipeline. 
```python
deployment.yaml = app.to_yaml()

# Structured deployment CLI
serve deploy deployment.yaml
```

## Compatibility, Deprecation, and Migration Plan
An important part of the proposal is to explicitly point out any compability implications of the proposed change. If there is any, we should thouroughly discuss a plan to deprecate existing APIs and migration to the new one(s).

- Ray Core
  - Serve Pipeline is co-designed with Ray Unified DAG API, where each DAG is always authored using Ray DAG API first.
  - The only new API introduced is `.bind()` method on ray decorated function or class.  
- Ray Serve 
  - Serve Pipeline DAG is transformed from Ray DAG where classes used are replaced with serve `Deployment` and class instances with deployment's `RayServeHandle` for better compatibility, deprecation as well as migration.
  - [Breaking Changes]
    - All args and kwargs passed into class or function in Serve Pipeline needs to be JSON serializable, enforced upon `build()` call. 



## Test Plan and Acceptance Criteria
The proposal should discuss how the change will be tested **before** it can be merged or enabled. It should also include other acceptance criteria including documentation and examples. 

## (Optional) Follow-on Work
Optionally, the proposal should discuss necessary follow-on work after the change is accepted.
