# Execution Optimizer for Ray Datasets

## Summary

Build the breakthrough foundation to tackle a series of fundamental issues around Ray Data. The foundation is (1) **lazy execution**, (2) **optimizer**, and (3) **vectorized execution with data batch**.

### General Motivation
Two key ML workloads for Datasets are (1) **data ingest**, where trainer processes (e.g., PyTorch workers), read, preprocess, and ingest data, and (2) **batch inference**, where we want to apply a pretrained model across a large dataset to generate predictions.

Currently Ray Data does not work out-of-box for both workloads with followed reasons:
* Data batch size and parallelism is challenging to tune by users, often resulting in poor performance and frustration.
* Expensive data conversion, copy, and materialization to object store, can happen everywhere.
* Lack of optimizer for execution plan, caused too much hand-holding with users to write the best program by hand.

To resolve the above issues, we need to change Ray Datasets in multiple perspectives:
* User experience:
  * Configurations auto-tuning: auto-tuning critical configurations such as data batch size and parallelism. Fewer configurations and less confusing for users.
  * Minimize cognitive load: lazy-execution first and only.
* Performance
  * Memory efficiency: zero-copy batch iterator. Operators pushdown. Eliminate unnecessary batch copy and format conversion.
* Extensibility
  * Easier to develop other optimization in the future
    * Special optimizer for AIR workload.
    * Example: prune unused columns through the execution plan as early as possible (as ColumnPruningPushDownRule).
* Simplify code base
  * Unify `LazyBlockList` and `BlockList`, delete `StatsActor`, eager mode and other legacy behavior.
  * Able to support more Datasets APIs.

### Should this change be within `ray` or outside?

The proposed change is inside the code of Ray Datasets, which is already part of the Ray project repository (“ray.data”).

## Stewardship
### Required Reviewers

@clarkzinzow, @jianoaix, @ericl, @stephanie-wang

### Shepherd of the Proposal (should be a senior committer)

@ericl

## Design and Architecture

Architecture after REP:

<img width="945" alt="new-architecture" src="https://user-images.githubusercontent.com/4629931/207807703-bb65db63-41a0-41d9-8e7b-154e1a0ed565.png">

Architecture before REP:

<img width="930" alt="old-architecture" src="https://user-images.githubusercontent.com/4629931/207808050-c07e51ed-ece4-4781-9f97-5a06ac107973.png">

### 1. Enable lazy execution mode by default

#### 1.1. How do things work today?

Ray Datasets has two execution modes:
* Eager mode
  * Each operation is executed immediately (i.e. eagerly).
* Lazy mode
  * Each operation is not executed immediately, but being added to the execution plan (i.e. lazily). Upon user calling consumption/action APIs (e.g. `fully_executed()`, `iter_batches()`, `take()` etc), all operations are executed together.

See https://docs.ray.io/en/latest/data/dataset-internals.html#lazy-execution-mode for more information.

Ray Datasets support both `Dataset` and `DatasetPipeline`.
* `Dataset` is using eager mode by default. Lazy mode can be enabled by calling `Dataset.lazy()`.
* `DatasetPipeline` is using lazy mode by default.

#### 1.2. Proposed Change

Change execution mode to lazy execution mode by default.

Eager mode has the benefit when users are doing ad-hoc experiments or debugging. Users can call each operation line by line, to experiment or debug. However, eager mode has no benefit over lazy mode, when coming to production workload (non ad-hoc, running whole Python program in real cluster), with followed reasons:

* When running the whole Python program, users do not debug the program line by line interactively.
* Under eager mode, each operation is forced to be executed immediately upon calling. Optimizations to the whole execution plan are often impossible. This often leads to suboptimal performance.

Lazy mode has proved to be effective and is the de-facto industry standard in the big data processing world (Spark, Presto/Trino, Hive). The major motivation with lazy mode is to build a plan optimizer to generate the optimal execution plan. It opens the door for more advanced techniques such as adaptive query execution, cost-based optimizer, pluggable execution back-end, etc. Lazy mode is the first step of the execution optimizer. Execution optimizer is the route to get us to optimal performance.

### 2. Introduce execution plan optimizer

#### 2.1. How do things work today?

* `Dataset` has `ExecutionPlan`. 
* `ExecutionPlan` has `BlockList` (as input) and a list of `Stage`s (as execution). 
* When executing `ExecutionPlan`, `ExecutionPlan._optimize()` is called to get an optimal list of `Stage`s. 
* After `_optimize()` is done, each `Stage` is executed one by one sequentially. First `Stage` reads input `BlockList`. All other `Stage` reads the output `BlockList` from the previous `Stage`, which is materialized in the Ray Object Store. 
* `ExecutionPlan` supports partial execution (so-called snapshot), i.e. only executing some of `Stage`s but not all.

#### 2.2. Proposed Change

The current design works for now, but has a lot of room for improvement. Without a proper plan/query optimizer, it’s hard to extend and engineeringly impossible to maintain cleanly.

##### 2.2.1. Interfaces

```python
class Optimizer():
  """Abstract class for optimizer.

  An optimizer transforms the plan, with a list of optimization rules and input statistics.
  """

  rules: List[Rule]

  def optimize(plan: Plan, statistics) -> Plan:
    for rule in rules:
      plan = rule.apply(plan, statistics)
    return plan 

class LogicalOptimizer(Optimizer):
  """Optimizer for logical plan.

  An optimizer transforms the logical plan, with a list of optimization Rules and input statistics.
  """

  rules: List[Rule]

  def optimize(plan: LogicalPlan, statistics) -> LogicalPlan:
    for rule in rules:
      plan = rule.apply(plan, statistics)
    return plan 

class PhysicalOptimizer(Optimizer):
  """Optimizer for physical plan.

  An optimizer transforms the physical plan, with a list of optimization Rules and input statistics.
  """

  rules: List[Rule]

  def optimize(plan: PhysicalPlan, statistics) -> PhysicalPlan:
    for rule in rules:
      plan = rule.apply(plan, statistics)
    return plan 

class Rule():
  """Rule to optimize plan.

  A rule to optimize logical or physical plan. The rule should be cost-based and generate the plan with minimal cost.  
  """
  
  def apply(plan: Plan, statistics) -> Plan:
    # specific optimization logic in each rule
```

##### 2.2.2. Milestones

* Step1: Introduce `Optimizer` class.
  * `ExecutionPlan._optimize()` should be refactored as a separate `Optimizer` class. `Optimizer` should be implemented as rule-based, and takes statistics into consideration (both file statistics and run-time execution statistics). So we are set out to build a cost-based optimizer with adaptive query execution.
  * `Rule` is a single piece of logic to transform a `Plan` into another `Plan` based on heuristics and statistics.
  * `Optimizer` runs all `Rule`s one by one to get the final `Plan`. Each `Rule` takes `Plan` as input, and generates a better `Plan`.
  * `Optimizer` should have two subclasses: `LogicalOptimizer` and `PhysicalOptimizer` for `LogicalOperator` and `PhysicalOperator` separately (more details below).
    * `LogicalOptimizer`, `PhysicalOptimizer`
    * `LogicalPlan`, `PhysicalPlan`
    * `LogicalOperator`, `PhysicalOperator`

* Step 2: Introduce `Operator` class.
  * Currently processing logic is modeled as `Stage` (e.g. read stage, map_batches stage, etc). The naming is confused when stages can be fused together into one stage, as each one no longer represents a stage boundary during execution. We should model `Stage` as an `Operator` class instead.
  * `Operator` should have two subclasses: `LogicalOperator` and `PhysicalOperator`.
    * `LogicalOperator`: usually one-to-one mapping to the same `Dataset` API. The `LogicalOperator` specifies what to do, but NOT specify how to do the operation. For example, `ParquetReadOperator` specifies to read Parquet files, but not specify how to read the Parquet files.
    * `PhysicalOperator`: mapping to a `LogicalOperator` (or part of a `LogicalOperator`). The `PhysicalOperator` specifies how to do the operation. For example, `ArrowParquetReadOperator` specifies to read Parquet files with the Arrow reader. One `LogicalOperator` can have more than one `PhysicalOperator` for it.
  * `Operator` depends on its children operators as its input. A single operator can depend on more than one operator. So the `Plan` consists of a DAG of `Operator`s (so-called query plan tree or execution graph in other systems).

* Step 3: Refactor `ExecutionPlan` to work with `Optimizer` and `Operator`.
  * `ExecutionPlan` takes a DAG of `Operator`s instead of a chaining of `Stage`s.
  * `Plan` should have two subclasses: `LogicalPlan` and `PhysicalPlan`.

* Step 4: Introduce new `Optimizer` rules for configurations auto-tuning
  * `AutoParallelismAndBatchSizeRule`: automatically decide batch size and read tasks parallelism, based on cluster resource, input data size, and previous execution statistics.
  * `Operator`s push down rules:
    * `BatchSizePushDownRule`: push down batching as early as possible in plan.
    * `BatchFormatPushDownRule`: push down batch_format transformation as early as possible in plan.
    * `LimitPushDownRule`: push down limit as early as possible in plan, up to data source (ideally just read N number of rows from input).
    * `ProjectionPushDownRule`: prune unused columns through plan as early as possible, up to datasource (ideally just read X columns from input, push SQL query into DB data source).

### 3. Zero-copy vectorized execution with data batch

#### 3.1. How things work today?

* Each `Stage` in `ExecutionPlan` is executed sequentially. 
* Each `Stage` materializes output as a list of `Block`s into the Ray Object Store, and this is served as input `Block`s for the next `Stage`. 
* If multiple `OneToOneStage`s fused together into one `Stage`, each sub-stage still executed sequentially, with each sub-stage output `Iterable[Block]` in worker heap memory. The final sub-stage of the fused `Stage` materializes the `Block`s into the Ray Object Store. 
* The read `Stage` can produce an `Iterable[Block]` with dynamic block splitting.

Example:

```
Read() -> Map_batches(foo, batch_format="numpy", batch_size=N) -> Map_batches(bar, batch_format="numpy", batch_size=M)
```

Data flow:

<img width="794" alt="data-flow-before" src="https://user-images.githubusercontent.com/4629931/207812215-db11f369-ef65-4e36-8070-0b4c4d46d2c9.png">

Each in-memory Block is 512MB by default. The good thing here is we loosely follow the [Volcano iterator model](https://paperhub.s3.amazonaws.com/dace52a42c07f7f8348b08dc2b186061.pdf) + [vectorized execution](https://www.vldb.org/pvldb/vol11/p2209-kersten.pdf) (same design choice as [Databricks Photon](https://www.databricks.com/product/photon) and [Meta Velox](https://engineering.fb.com/2022/08/31/open-source/velox/)). The bad thing is to enforce `Block` size as 512MB, which may not work well with `batch_size` in `map_batches`. In `map_batches`, multiple `Block`s may be bundled or coalesced into one `Batch`, or one `Block` may be sliced into multiple `Batch`es. Also note, there's unnecessary data format conversion between NumPy batch and Arrow block. We should minimize the data copy and format conversion for intermediate in-memory data.

#### 3.2. Proposed Change

If users do not set `batch_size` in `map_batches`, we should choose a `batch_size `dynamically based on cluster resource, input size, estimated in-memory data size, etc. If users set `batch_size`, we respect the user provided `batch_size`.

Use the `batch_size` from `map_batches`, to determine the `batch_size` in read stage. So each `Block` output by read is a batch directly passed to `map_batches` with zero-copy. The final output `Block`s to Object store can target 512MB though (may incur a copy for final output `Block`s).

Data flow:

<img width="795" alt="data-flow-after" src="https://user-images.githubusercontent.com/4629931/207812234-d94a6683-5e0d-4eb2-a7c3-6f7bb656f2b2.png">

The new proposal would eliminate that batch to block copy and pass the batch directly to the following stage.

#### 3.2.1. Interfaces

NOTE: `OneToOneOperator` used here is the same as `OneToOneOperator` in "Native pipelining support in Ray Datasets" REP.

```python
class BatchedOperator(OneToOneOperator):
  """Abstract class for batched OneToOneOperator.

  BatchedOperator should implement vectorized execution on input data blocks, in batched manner.
  """

  def execute_one(blocks: Iterator[Block], metadata: Dict[str, Any]) -> Iterator[Block]
    input_batches = convert_block_to_batch(blocks)
    output_batches = process_batches(input_batches)
    return convert_batch_to_block(output_batches)
  
  def process_batches(input: Iterator[Batch]) -> Iterator[Batch]:
    # Process each Batch from input


class ArrowParquetReadOperator(BatchedOperator):
  """Operator to read Parquet file in Arrow format.
  
  ArrowParquetReadOperator uses PyArrow Reader to read Parquet files into Arrow batches.
  """

  def process_batches(input: Iterator[Batch]) -> Iterator[Batch]:
    for file_name in input:
      with open(file_name) as f:
        for batch in f.to_batches():
          yield batch

class MapBatchesOperator(BatchedOperator):
  """Operator for map_batches() to run batch UDF on input data."""

  batch_size: int
  batch_fn: BatchUDF

  def process_batches(input: Iterator[Batch]) -> Iterator[Batch]:
    for new_batch in batcher(input, batch_size)
      yield batch_fn(new_batch)
 
class WholeStageFusedOperator(BatchedOperator):
  """Operator to fuse multiple batched operators together into one operator, which is executed in the same process.
  """

  fused_operators: List[BatchedOperator]

  def process_batches(input: Iterator[Batch]) -> Iterator[Batch]:
    output = fused_operators[0].process_batches(input)
    for op in fused_operators[1:]:
      output = op.process_batches(output)
    return output
```

Corner cases:
* When users set different `batch_size` for multiple `map_batches()`. We can choose (1). respect users setting (will incur copy between `map_batches`), or (2). decides a min/max `batch_size` and uses the same `batch_siz`e across `map_batches`.
  * Filter can happen to reduce rows
  * Image rotation can change number of images
  * Solution: have a `Batcher` to buffer batches for different `batch_size`s specified in `map_batches`.
* When the user's `map_batches()` function changes the number of rows in a batch, and has multiple `map_batches()`. We can choose (1). respect users settings, and aim to produce a batch with `batch_size` for the following `map_batches()`, or (2).do nothing for the output batch, feed directly into following `map_batches()`.

### 4. Benchmark

We created a simple prototype ([PR](https://github.com/ray-project/ray/pull/30962)), to avoid data copy and buffering in `map_batches()`. This mimics the final behavior of zero-copy vectorized execution with data batch. We evaluated the performance against the current master branch in two input data. The evaluation benchmark code can be found in this [PR](https://github.com/ray-project/ray/pull/30961).

On a single-node environment (m5.4xlarge), the prototype is more than 2x faster than the current master branch. The major reason is to avoid intermediate data copy and materialization between stages. The input data is NYC taxi dataset on year 2018 (Parquet format), and batch format is Pandas. The `map_batche`s UDF is a dummy identity function `lambda x: x`, to make sure our benchmark is focusing on dataset internal overhead, instead of cost of user code.

In the benchmark, we have 3 different runs:
* "default" run: the current master branch.
* "lazy" run: the current master branch with lazy execution enabled.
* "prototype" run: prototype code with lazy execution enabled.

The first evaluation is to run the same `map_batches()` calls with different `batch_size`. There are two `map_batches()` calls here. As below, we can see the latency for all runs is increased as `batch_size` is decreased. This is expected as more intermediate small batches are generated, so more overhead is incurred during execution. The "lazy" run is better than the "default" run because of avoiding intermediate data materialization to the Ray object store. The "prototype" run is better than "lazy" run because of avoiding intermediate data copy when generating a batch from a data block, and vice versa.

<img width="783" alt="first-evaluation" src="https://user-images.githubusercontent.com/4629931/207815405-22b72957-9e39-4355-ba00-4438b58e6e65.png">

The second evaluation is to run the same `map_batches()` calls multiple times with the same `batch_size`. As below, we can see the latency for all runs is increased as the number of calls is increased. This is expected as more function calls incur more overhead. The "default" runs and "lazy" runs are taking much longer time than the "prototype" runs, with the same reason above.


<img width="779" alt="second-evaluation" src="https://user-images.githubusercontent.com/4629931/207815413-7264ef25-7971-4f75-89c7-b6c610fc5194.png">

## Compatibility, Deprecation, and Migration Plan

Since the changes are mostly internal, we should be able to migrate to the new backend without any breaking changes to the Ray Dataset APIs. The migration plan is as follows:
* [Ray 2.3] Lazy-execution is enabled by default.
* [Ray 2.3] The new `Optimizer` and `Operator` API is implemented and merged.
* [Ray 2.3/2.4] Implement each `Stage` logic as `LogicalOperator` and `PhysicalOperator` separately for read, map_batches, etc.
* [Ray 2.3/2.4] Existing optimization logic is ported to as `Optimizer` `Rule` one by one. Implement new optimization for configuration auto-tuning.
* [Ray 2.4] We complete the testing and validation of `Optimizer` and `Operator`, and change to use the `Operator` code path via a switch in `DatasetContext`.
* [Ray 2.4/2.5] Remove `Stage` legacy code path after completely rolling out `Optimizer` and `Operator`.

## Test Plan and Acceptance Criteria

* Work seamlessly with REP "Native pipelining support in Ray Datasets"
* Unit tests to check the algorithmic logic of `Optimizer` and all `Operator`s.
* Passing all existing unit and release tests for Ray Data.
* Performance testing for new `Optimizer` rules and `Operator`s implementation with current AIR benchmarks and users-provided workload. Note the benchmark and prototype work should happen before checking in the real implementation.

## (Optional) Follow-on Work

This REP impacts ongoing REP "Native pipelining support in Ray Datasets". We may need to refine the `Optimizer` and `PhysicalOperator` API proposal to align needs, though we do not anticipate major changes are necessary.
