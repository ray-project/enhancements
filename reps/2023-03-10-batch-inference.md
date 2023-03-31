# Revamping APIs for Offline Batch Inference

## Summary
Reduce the friction for batch inference of pre-trained models by updating the APIs to do the following:
1. Remove AIR Checkpoints from the critical path for batch prediction.
2. Simpler interface for chaining multiple preprocessing and prediction stages.

### General Motivation
There are changes to Ray Datasets in 2.3/2.4:
1. Ray Datasets became lazy by default in Ray 2.3. Execution is no longer triggered until consumption is explicitly called.
2. In Ray 2.4, Ray Datasets will move to streaming execution by default, resulting in there no longer being a separation between `Dataset` and `DatasetPipeline`.

With these changes, certain API choices that were made for the AIR batch prediction API are no longer the most optimal for simplicity and usability.
1. Previously, AIR would manually fuse preprocessing and prediction stages, requiring `BatchPredictor` to be aware of the preprocessors passed in via a `Checkpoint`. This is not needed anymore with Datasets lazy by default, as the fusion logic can be pushed down to the Datasets layer.
2. Previously, the separation of `Dataset` and `DatasetPipeline` resulted in `BatchPredictor` needing 2 analgous APIs: `predict` and `predict_pipelined`. This is not needed anymore with streaming as default execution.

### Should this change be within `ray` or outside?
main `ray` project. Changes are made to Ray Data and Ray AIR level.

## Stewardship
### Required Reviewers
The proposal will be open to the public, but please suggest a few experience Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@ericl, @clarkzinzow, @c21, @gjoliver

### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

@ericl

## Design and Architecture

## User facing API Changes

1. Preprocessors can be directly passed to `Dataset.map_batches`.

```python
ds = ray.data.read_images(...)
preprocessor1 = BatchMapper(lambda x: x+1)
preprocessor2 = TorchVisionPreprocessor(torchvision.transforms.Crop(224, 224))
ds.map_batches(preprocessor1).map_batches(preprocessor2)
```

2. Predictors can be directly passed to `Dataset.map_batches`

```python
ds = ray.data.read_images(...)
model = resnet50()
predictor = TorchPredictor(model=model)

# Alternatively, create from AIR Checkpoint.
# predictor = TorchPredictor.from_checkpoint(air_checkpoint)

predictions = ds.map_batches(predictor, compute=ray.data.ActorPoolStrategy(size=8))
```

Exception is raised if ActorPoolStrategy is not specified.

```python
ds = ray.data.read_images(...)
model = resnet50()
predictor = TorchPredictor(model=model)
predictions = ds.map_batches(predictor) # Fails with ValueError.
```

3. `feature_columns` and `keep_columns` are added to the base `Predictor.predict` interface.

```python
def predict(self, 
    data: DataBatchType, 
    feature_columns: Optional[List[str]] = None,
    keep_columns: Optional[List[str]] = None,
    **kwargs) -> DataBatchType:
    """Perform inference on a batch of data.

    Args:
        data: A batch of input data of type ``DataBatchType``.
        feature_columns: List of columns in the preprocessed dataset to use for
                prediction. Columns not specified will be dropped
                from `data` before being passed to the predictor.
                If None, use all columns in the preprocessed dataset.
        keep_columns: List of columns in the preprocessed dataset to include
            in the prediction result. This is useful for calculating final
            accuracies/metrics on the result dataset. If None,
            the columns in the output dataset will contain
            just the prediction results.
        kwargs: Arguments specific to predictor implementations. These are passed
            directly to ``_predict_numpy`` or ``_predict_pandas``.

    Returns:
        Prediction result. The return type will be the same as the input type.
    """
```

Users can specify these through `fn_args` and `fn_kwargs` in `map_batches`

```python
ds = ray.data.read_images(s3://images_with_labels)
model = resnet50()
predictor = TorchPredictor(model=model)
predictions_with_labels = ds.map_batches(
    predictor, 
    compute=ray.data.ActorPoolStrategy(size=8),
    fn_kwargs={"feature_columns": ["images"], "keep_columns": ["label"]}
)
assert "predictions" in predictions_with_labels.schema().names
assert "label" in predictions_with_labels.schema().names
```
      
### Example: Multi-stage Inference Pipeline from pre-trained model

Note that Checkpoints are not on the critical path, but can still be used.

#### Before
```python
preprocessor1 = BatchMapper(lambda x: x+1)
preprocessor2 = TorchVisionPreprocessor(torchvision.transforms.Crop(224, 224))
chained = Chain([preprocessor1, preprocessor2])

model = pretrained_resnet()
checkpoint = TorchCheckpoint.from_model(model, preprocessor=chained)
batch_predictor = BatchPredictor.from_checkpoint(checkpoint)

ds = ds.read_parquet("s3://data")
predictions = batch_predictor.predict(ds)
```

#### After
```python
model = pretrained_resnet()
predictor = TorchPredictor(model)

# Alternatively, create from AIR Checkpoint.
# predictor = TorchPredictor.from_checkpoint(air_checkpoint)

predictions = ds.read_parquet("s3://data")
    .map_batches(lambda x: x+1)
    .map_batches(TorchVisionPreprocessor(torchvision.transforms.Crop(224, 224)))
    .map_batches(predictor, compute=ray.data.ActorPoolStrategy(size=8))
```

### Example: Implementing a custom Predictor.
Using Checkpoints are no longer necessary.

#### Before
```python
class CustomPredictor(Predictor):
    def __init__(
      self,
      net: gluon.Block,
      preprocessor: Optional[Preprocessor] = None,
    ):
        self.net = net
        super().__init__(preprocessor)

    @classmethod
    def from_checkpoint(
        cls,
        checkpoint: Checkpoint,
        net: gluon.Block,
    ) -> Predictor:
        with checkpoint.as_directory() as directory:
            path = os.path.join(directory, "net.params")
            net.load_parameters(path)
        return cls(net, preprocessor=checkpoint.get_preprocessor())

    def _predict_numpy(
        self,
        data: Union[np.ndarray, Dict[str, np.ndarray]],
        dtype: Optional[np.dtype] = None,
    ) -> Dict[str, np.ndarray]:
        # If `data` looks like `{"features": array([...])}`, unwrap the `dict` and pass
        # the array directly to the model.
        if isinstance(data, dict) and len(data) == 1:
            data = next(iter(data.values()))

        inputs = mx.nd.array(data, dtype=dtype)
        outputs = self.net(inputs).asnumpy()

        return {"predictions": outputs}

dataset = ray.data.read_images(
    "s3://anonymous@air-example-data-2/imagenet-sample-images", size=(224, 224)
)


def preprocess(batch: Dict[str, np.ndarray]) -> Dict[str, np.ndarray]:
    # (B, H, W, C) -> (B, C, H, W)
    batch["image"] = batch["image"].transpose(0, 3, 1, 2)
    return batch


# Create the preprocessor and set it in the checkpoint.
# This preprocessor will be used to transform the data prior to prediction.
preprocessor = BatchMapper(preprocess, batch_format="numpy")
checkpoint.set_preprocessor(preprocessor=preprocessor)


net = gluon.model_zoo.vision.resnet50_v1(pretrained=True)
predictor = BatchPredictor.from_checkpoint(
    checkpoint, MXNetPredictor, net=net
)
predictor.predict(dataset)
```

#### After
```python
class MXNetPredictor(Predictor):
    def __init__(
      self,
      net: gluon.Block,
    ):
        self.net = net
        super().__init__()

    def _predict_numpy(
        self,
        data: Union[np.ndarray, Dict[str, np.ndarray]],
        dtype: Optional[np.dtype] = None,
    ) -> Dict[str, np.ndarray]:
        # If `data` looks like `{"features": array([...])}`, unwrap the `dict` and pass
        # the array directly to the model.
        if isinstance(data, dict) and len(data) == 1:
            data = next(iter(data.values()))

        inputs = mx.nd.array(data, dtype=dtype)
        outputs = self.net(inputs).asnumpy()

        return {"predictions": outputs}

dataset = ray.data.read_images(
    "s3://anonymous@air-example-data-2/imagenet-sample-images", size=(224, 224)
)

def preprocess(batch: Dict[str, np.ndarray]) -> Dict[str, np.ndarray]:
    # (B, H, W, C) -> (B, C, H, W)
    batch["image"] = batch["image"].transpose(0, 3, 1, 2)
    return batch

predictor = MXNetPredictor(gluon.model_zoo.vision.resnet50_v1(pretrained=True))
dataset.map_batches(preprocess, batch_format="numpy").map_batches(predictor_creator, compute=ray.data.ActorPoolStrategy(size=8))
```

## Internal interfaces & implementation details
1. Introduce a DeveloperAPI `BatchMappable` interface that `Preprocessor` and `Predictor` subclass. This interface will define the following:
- Logic for the actual computation (either prediction or preprocessing)
- Validating the user provided batch_format
- Validating the user provided compute strategy

```python
@DeveloperAPI
class BatchMappable(Callable[[T], U]):
   """Subclass this if you want your class to be passable to Dataset.map_batches."""
   
    def __call__(self, batch: T) -> U:
        """Transform the provided batch.

        Args:
            batch: The batch to transform.
        """
        
        raise NotImplementedError

    def setup(self):
        """Additional setup logic to run.

        If a callable class is being used, this function will be run in the `__init__`
        of the Callable class.
        """
        pass
       
    def validate_batch_format(self, batch_format: Optional[str]) -> Optional[str]:
        """Validate that the provided `batch_format` is compatible with this `BatchMappable`.

        Raises an error if the provided `batch_format` is not supported with this `BatchMappable` 
        (for example Pyarrow format for Predictors.)

        If `batch_format` is "default", returns the preferred `batch_format` for this `BatchMappable`.

        Args:
            batch_format: The `batch_format` that is provided to `map_batches`.

        Returns:
            The batch format to use.
        """
       return None

    def validate_compute_strategy(self, strategy):
        """Validate that the provided `strategy` is compatible with this `BatchMappable`.

        For example, enforce that `ActorPoolStrategy` is provided for Predictors.
        """
        pass

    def use_callable_class(self) -> bool:
        """Indicates whether this `BatchMappable` should use a callable class."""
```

`map_batches` will support instances of `BatchMappable` as a supported UDF

2. Add a private `to_checkpoint` serializer for `Predictor`, analagous to the current `from_checkpoint` deserializer. 

```python
class Predictor:
    def _to_checkpoint(self) -> Checkpoint:
        """Create a serializable Checkpoint from this Predictor.

        Returns:
            Checkpoint containing the state for this Predictor.
        """
    
    def __getstate__(self):
        return self._to_checkpoint()

    def __setstate__(self, checkpoint):
        # Same logic as `from_checkpoint`.

    def use_callable_class(self) -> bool:
        return True
```

3. When using Predictors with AIR Checkpoints, move the construction logic to `setup` so that it can be called within the actor.

```python
class Predictor:
    def _to_checkpoint(self) -> Checkpoint:
        """Create a serializable Checkpoint from this Predictor.

        Returns:
            Checkpoint containing the state for this Predictor.
        """
        ...
    
    def setup(self):
        if self._checkpoint:
            # Initialize the state from the Checkpoint.
    
    @classmethod
    def from_checkpoint(cls, checkpoint):
        return Predictor(_checkpoint=checkpoint)

    def use_callable_class(self) -> bool:
        return True
```

Putting this all together, the implementation for `map_batches` will look like the following

```python
def map_batches(
        self,
        fn: BatchUDF,
        *,
        batch_size: Optional[Union[int, Literal["default"]]] = "default",
        compute: Optional[Union[str, ComputeStrategy]] = None,
        batch_format: Optional[str] = "default",
        zero_copy_batch: bool = False,
        fn_args: Optional[Iterable[Any]] = None,
        fn_kwargs: Optional[Dict[str, Any]] = None,
        fn_constructor_args: Optional[Iterable[Any]] = None,
        fn_constructor_kwargs: Optional[Dict[str, Any]] = None,
        **ray_remote_args,
    ) -> "Dataset[Any]":

    if isinstance(fn, BatchMappable):
        fn.validate_compute_strategy(compute)
        batch_format = fn.validate_batch_format(batch_format)

        if fn.use_callable_class:
            # Put the BatchMappable in the object store once.
            batch_mappable_ref = ray.put(fn)

            class _Wrapper:
                def __init__(self, batch_mappable_ref: ObjectRef[BatchMappable]):
                    self.batch_mappable = ray.get(batch_mappable_ref)
                
                def __call__(self, *args, **kwargs):
                    return self.batch_mappable(*args, **kwargs)
            
            fn = _Wrapper
        
        # Rest of the implementation stays the same
        ...
```

## FAQ
1. If I pass in a `Predictor` directly, won't initiliazation happen for every batch?

No. `Predictors` will require the use of a Callable Class, and so we wrap the Predictor in a Callable class internally, like in the example above.

2. If I have an AIR Checkpoint, won't this require initialization on the driver?

No. If a Predictor is created from an AIR Checkpoint, we delay the initialization to a `setup` call that happens only in the 
actor.

## Compatibility, Deprecation, and Migration Plan
An important part of the proposal is to explicitly point out any compability implications of the proposed change. If there is any, we should thouroughly discuss a plan to deprecate existing APIs and migration to the new one(s).

Ray 2.5: Onboard new users onto new APIs
- Support Preprocessors and Predictors in `map_batches`.
- Rewrite all batch inference examples to use this API, and use `map_batches` directly instead of `BatchMapper` and `Chain`.
- `BatchPredictor` will still be fully supported.

Ray 2.6: Start migrating existing users to new APIs
- Soft Deprecate `BatchPredictor`. Warn if this API is used, and remove from Documentation.
- Add migration guide in documentation for guidance on how to migrate to the new APIs.

Ray 2.7: Deprecate old APIs
- Hard deprecate `BatchPredictor`.

## Test Plan and Acceptance Criteria
The proposal should discuss how the change will be tested **before** it can be merged or enabled. It should also include other acceptance criteria including documentation and examples. 

- Unit and integration for new APIs
- Documentation and examples on new API.
