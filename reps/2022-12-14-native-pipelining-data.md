
## Summary - Native pipelining support in Ray Datasets
### General Motivation
Two key ML workloads for Ray Datasets are (1) **data ingest**, where trainer processes (e.g., PyTorch workers), read, preprocess, and ingest data, and (2) **batch inference**, where we want to apply a pretrained model across a large dataset to generate predictions.

Both workloads are performance sensitive, due to the need to maximize GPU utilization (e.g., may require multiple GB/s throughput per worker). Currently, Dataset users are generally successful when operating on small to medium datasets (e.g., 1-100GB), that fit comfortably in Ray object store memory. However, users often struggle to configure Ray's DatasetPipeline (pipelined data loading) to operate efficiently on larger-than-memory Datasets. In addition, certain aspects of its execution model, such as recreating actor pools for each window, add significant overheads.

This REP proposes a pipelined-first Datasets execution backend that improves efficiency and removes the need to carefully tune the configuration of DatasetPipeline. After this proposal, it will be possible to deprecate and remove DatasetPipeline, and also to remove the "pipelined" variants of AIR APIs for Training and Inference.

### Should this change be within `ray` or outside?
The proposed change is to the execution backend of Ray Datasets, which is already part of the Ray project repository.

## Stewardship
### Required Reviewers
The proposal will be open to the public, but please suggest a few experience Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@c21, @clarkzinzow, @jianoaix, @stephanie-wang

### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

@stephanie-wang

## Design and Architecture

### Challenges 

Datasets currently executes operations using the bulk synchronous parallel (BSP) execution model. However, BSP has fundamental limitations when it comes to key ML workloads:

1. **Iterating over a Dataset is memory-inefficient in BSP**. This is since the Dataset becomes fully materialized in memory when bulk executed. Pipelined / streaming data loading and processing is desirable to reduce memory usage and avoid spilling for large Dataset ingest jobs.

2. **Heterogeneous pipelines materialize intermediate data**. For pipelines that are disaggregated or involve both CPU and GPU stages, data between stages is materialized in cluster memory, reducing performance.

To avoid these limitations, Ray 2.0 featured a DatasetPipeline class that pipelines micro-batched Dataset executions. However, users have found it hard to configure the pipelining settings in practice. This is because the pipeline "window size" config affects both available parallelism and memory usage simultaneously. In addition, there is an efficiency gap (see Benchmarks).
                                           
### Proposal

Support pipelined execution natively in Datasets, moving away from the bulk execution model by default. This is mostly an internal execution model change; the Datasets API remains largely unchanged. After this proposal, it will be possible to deprecate and remove DatasetPipeline.

Thanks to the flexibility of Ray, this change is not as large as it may seem. For example, Ray's lineage fault tolerance will continue to "just work" even in the pipelined execution model. The main challenge is defining a clear set of internal interfaces to make the migration incremental.

#### Corner Cases

There are a few APIs (i.e., the variations of .split()) that are challenging to pipeline. This is because for some datasources (e.g., csv, json), we do not know ahead of time how many records are in each file, hence we need to fully execute read tasks to perform exact splits.

The short term solution is to treat split() as a special case, preserving backwards compatibility and logging a warning to the user that this triggers synchronous execution for certain datasources. We can add a new iter_split() API that does support pipelining in all use cases (but that has slightly different semantics from split()).

### Interfaces

We introduce a new interface between the operator and execution layers of Datasets, which enables flexible selection of the execution strategy (bulk vs pipelined). The operator layer defines "what" to execute (e.g., Arrow table operations), and the executor defines "how" it is executed (e.g., bulk vs pipelined). Previously, operator definition and execution were intertwined in the code.
![graphs](https://user-images.githubusercontent.com/14922/207695244-b5a90d67-bdc9-4519-9bf1-fa01ecb877a4.png)

For example, suppose you are running the following "Two-stage inference example":

```python
	ray.data.read_parquet()
		.map_batches(fn_1)
		.map_batches(fn_2)
		.map_batches(infer, compute="actors", num_gpus=1)
		.write_parquet()
```

Datasets translates the user API calls into an optimized series of operators, each of which should be run in a separate task:

```
	Read() -> Map([read, fn_1, fn_2]) -> Map(infer, GPU) -> Map(write)
```

The Executor is responsible for deciding how to execute the final operator DAG. It could execute the operators one at a time (bulk synchronous execution), or concurrently (pipelined execution). In the above example, the pipelined strategy would use much less peak memory, since it avoids fully materializing the data between the CPU and GPU stages.

#### Class Hierarchy

The proposed interface class hierarchy is as follows:
- **RefBundle**: A group of Ray object references and their metadata. This is a logical "Block" in the Dataset.
- **PhysicalOperator**: Node in the operator DAG that transforms one or more RefBundle input streams to a RefBundle output stream.
    * Example subclasses (these may be grouped into sub-abstractions such as OneToOneOperator and AllToAllOperator; the definition of these abstractions is outside the scope of this REP):
        + Read/Write
        + Map
        + Sort
        + RandomShuffle
        + ...
- **Executor**: Execute an operator DAG, returning an output stream of RefBundles.
    * Subclasses
        + BulkExecutor: Implements BSP execution of the DAG (for legacy testing).
        + PipelinedExecutor: Implements pipelined execution.


The formal interface of PhysicalOperator is as follows. This is the interface that the Executor interacts with to implement execution of the operator DAG. Each operator manages its own distributed execution state, i.e., tasks in flight and references to intermediate objects, but is controlled by the Executor, which supplies input RefBundles and consumes output RefBundles:

```python
class PhysicalOperator:
    """Abstract class for physical operators.

    An operator transforms one or more input streams of RefBundles into a single
    output stream of RefBundles.

    Operators are stateful and non-serializable; they live on the driver side of the
    Dataset execution only.
    """

    def __init__(self, name: str, input_dependencies: List["PhysicalOperator"]):
        self._name = name
        self._input_dependencies = input_dependencies
        for x in input_dependencies:
            assert isinstance(x, PhysicalOperator), x

    @property
    def name(self) -> str:
        return self._name

    @property
    def input_dependencies(self) -> List["PhysicalOperator"]:
        """List of operators that provide inputs for this operator."""
        assert hasattr(
            self, "_input_dependencies"
        ), "PhysicalOperator.__init__() was not called."
        return self._input_dependencies

    def get_stats(self) -> StatsDict:
        """Return recorded execution stats for use with DatasetStats."""
        raise NotImplementedError

    def __reduce__(self):
        raise ValueError("PhysicalOperator is not serializable.")

    def __str__(self):
        if self.input_dependencies:
            out_str = ", ".join([str(x) for x in self.input_dependencies])
            out_str += " -> "
        else:
            out_str = ""
        out_str += f"{self.__class__.__name__}[{self._name}]"
        return out_str

    def num_outputs_total(self) -> Optional[int]:
        """Returns the total number of output bundles of this operator, if known.

        This is useful for reporting progress.
        """
        if len(self.input_dependencies) == 1:
            return self.input_dependencies[0].num_outputs_total()
        return None

    def add_input(self, refs: RefBundle, input_index: int) -> None:
        """Called when an upstream result is available."""
        raise NotImplementedError

    def inputs_done(self, input_index: int) -> None:
        """Called when an upstream operator finishes."""
        pass

    def has_next(self) -> bool:
        """Returns when a downstream output is available."""
        raise NotImplementedError

    def get_next(self) -> RefBundle:
        """Get the next downstream output."""
        raise NotImplementedError

    def get_tasks(self) -> List[ray.ObjectRef]:
        """Get a list of object references the executor should wait on."""
        return []

    def notify_task_completed(self, task: ray.ObjectRef) -> None:
        """Executor calls this when the given task is completed and local."""
        raise NotImplementedError

    def release_unused_resources(self) -> None:
        """Release any currently unused operator resources."""
        pass
```

This is how the "Two-stage inference example" breaks down into a DAG of operators:
![graphs](https://user-images.githubusercontent.com/14922/207695905-8054e644-0600-4a55-a266-2fc0a9396223.png)

As another example, suppose we replaced the `Map(infer, GPU)` stage with `Sort()` operator. This would generate a pipeline with a blocking operator (orange) that acts as a barrier prior to the actual sort:
![graphs](https://user-images.githubusercontent.com/14922/207696019-b37fab59-b49a-4819-98c0-d0d7646b68ba.png)

In this way, we can support both pipelineable and non-pipelineable operations.

### Execution Algorithm

BulkExecutor (for legacy support only) is implemented as follows:
1. The executor traverses the operator DAG in depth-first order starting from the input, executing operator dependencies synchronously and feeding their output to downstream operators.
2. When the traversal completes, the output RefBundles are returned as a list.

PipelinedExecutor is implemented as follows:
1. The executor determines a "parallelism budget" and "memory budget". By default, it uses all cores in the cluster and 50% of the object store memory. This can be overridden by the user and adjusted over time as the cluster scales up or down.
2. The executor traverses the DAG to build operator state for each operator. The operator state includes a list of input RefBundle queues, and an output RefBundle queue. The executor connects the input and output queues of operators together.
3. The executor performs the following next steps in a loop until execution completes:
     1. It processes any completed operator Ray tasks, putting their result onto the operator RefBundle queues.
     2. If the executor has remaining memory and parallelism budget, it selects an operator to run and dispatches a new task for the operator.

The algorithmic objective of PipelinedExecutor is to maximize pipeline throughput while remaining under its resource budget. We plan to make this operator selection algorithm pluggable. The initial implementation will use a simple heuristic that tries to balance the output queue sizes of each operator (intuitively, implementing simple backpressure).

When the output of PipelinedExecutor is an iterator (i.e., for data ingest), the PipedlinedExecutor can apply further optimizations such as co-locating the output data blocks with the physical node of the iterator consumer.

### Benchmarks

We prototyped a PipelinedExecutor implementation here: https://github.com/ray-project/ray/pull/30222 ("Datastream Prototype") and evaluated its performance against Datasets/DatasetPipeline on two workloads.

On a distributed data ingest workload (cluster of 8x 16-core machines), Datastream was more than 2x faster than a well-tuned DatasetPipeline implementation. This is likely because the "sliding window" execution improves CPU efficiency, reduces the impact of stragglers, and also reduces memory usage. Non-pipelined Datasets execution ground to a halt here due to excessive disk spilling.
![graphs](https://user-images.githubusercontent.com/14922/207696528-0c58c757-e704-44bb-961d-ff23864c344f.png)

On a two-stage batch inference workload (same cluster of 8x 16-core machines), Datastream was 20% faster than a well-tuned DatasetPipeline implementation. More importantly, it was also more robust. Adjusting the parallelism budget of Datastream between 20-100 changed its performance by <10%. In contrast, adjusting the DatasetPipeline window size by 2x impacted performance by >30%. Non-pipelined Dataset execution was much slower due to disk spilling.
![graphs](https://user-images.githubusercontent.com/14922/207696590-608be661-b8fd-4a1a-a0ab-34bc9c6644d3.png)

In summary, we validate the performance benefits of the proposed approach compared to both existing Dataset and DatasetPipeline abstractions. The source code of the benchmarks can be found here:
- https://gist.github.com/ericl/6de562785571c56dceb5bceafc7fcb71
- https://gist.github.com/ericl/a55bd5adcb319ca47cd079bcbf7a7cda
- https://gist.github.com/ericl/fa868d16a2c87505764b88668ec1c84b
- https://gist.github.com/ericl/02a825fd7279b6bf08364eab600a217e

### Related Systems

A few related systems from the ML and data space are:
- ML data loaders: These generally use a pipelined execution model, similar to the proposal here.
    * tf.data and torch.data: These implement framework-specific pipelines for training ingest. Ray Data serves a similar role for ingest, but is framework-agnostic and is designed to work well for scalable batch inference workloads as well.
    * tf.data service: Implements disaggregated data preprocessing. Ray Data provides the same functionality, and also supports advanced operations such as per-epoch shuffle.
    * Petastorm: implements data reading and preprocessing for parquet datasets ingest, in a framework-agnostic way.
- Spark RDDs: Implements bulk-synchronous distributed execution; similar to Datasets.
- Spark DStreams: Implements micro-batched execution; similar to DatasetPipeline.
- Flink / Cloud Dataflow / Presto / etc: Implements pipelined execution, similar to the proposal here; focus is on ETL and analytics.

After the pipeline-by-default change, Ray Data will be in a strong position relative to these other systems for ML data workloads: it will be the most flexible (supporting multiple frameworks, both ingest and inference, and advanced operators such as global shuffle), and will have better performance than before.

Implementation-wise, there are also a couple "novel" directions with the proposed design worth noting, mostly around leveraging Ray's capabilities:
- Use of fine-grained lineage for fault tolerance in a pipelined system.
- Use of a centralized controller / executor to schedule the pipelined execution.

### Alternatives
Alternatives include improving the configurability of DatasetPipeline, or adding a new Datastream class in addition to existing Dataset and DatasetPipeline classes. The former leaves some efficiency gains on the table, as seen in the Benchmarks section, and is likely more difficult for users to tune. The latter introduces more concepts, while not significantly reducing the scope of the engineering work.

## Compatibility, Deprecation, and Migration Plan
Since the changes are mostly internal, we should be able to migrate to the new backend without any breaking changes to the Ray Data APIs. The migration plan is as follows:
1. [Ray 2.3] Dataset is ported to use the new executor interface, but using BulkExecutor by default.
2. [Ray 2.3/2.4] We complete the testing and validation of PipelinedExecutor, and change the default executor for Datasets from BulkExecutor to PipelinedExecutor.
3. [Ray 2.4/2.5] Further improvements to bring feature parity with DatasetPipeline, which would enable us to deprecate and remove DatasetPipeline.
    - We may consider other alternatives such as just adding special Repeat / ShuffleWithWindow operators instead of faithfully reimplementing the original DatasetPipeline API.
4. [Optional] Rename Datasets to Datastream to clarify positioning.

## Test Plan and Acceptance Criteria
There are three prongs for testing the new execution backend:
1. Unit tests to check the algorithmic logic of PipelinedExecutor.
2. Passing all existing unit and release tests for Datasets / DatasetPipelines.
3. Performance tests to test the scalability of PipelinedExecutor (to a larger extent than the microbenchmark-style workloads shown above). Key workloads are the already-planned pipelined training and inference MLPerf benchmarks.

We should also port the high-level AIR APIs to take advantage of the performance improvements. This involves:
1. Removing the explicit "pipelined mode" from Ray Train and BatchPredictor. All training and inference data reads will be pipelined by default.
2. Updating the configuration of Ray Train to support Datastream configurations.
3. Updating the benchmarks and documentation.

## (Optional) Follow-on Work
This REP impacts ongoing work to revamp the code organization of the operator layer of Datasets. We may need to refine the PhysicalOperator interface to align needs, though we do not anticipate major changes are necessary.
