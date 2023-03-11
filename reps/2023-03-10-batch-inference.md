# Revamping APIs for Offline Batch Inference

## Summary
Reduce the friction for batch inference of pre-trained by updating the APIs to do the following:
1. Remove AIR Checkpoints from the critical path for batch prediction.
2. Simpler interface for chaining multiple preprocessing and prediction stages.

### General Motivation
There are changes to Ray Datasets in 2.3/2.4:
1. Ray Datasets became lazy by default in Ray 2.3. Execution is no longer triggered until consumption is explicitly called.
2. In Ray 2.4, Ray Datasets will move to streaming execution by default, resulting in there no longer being a separation between `Dataset` and `DatasetPipeline`. 
3. In Ray 2.4, Ray Datasets will support a `"zero_copy"` batch format, allowing the core Ray Data to decide the most optimal batch format to use.

With these changes, certain API choices that were made for the AIR batch prediction API are no longer the most optimal for simplicity and usability.
1. Previously, AIR would manually fuse preprocessing and prediction stages, requiring `BatchPredictor` to be aware of the preprocessors passed in via a `Checkpoint`. This is not needed anymore with Datasets lazy by default, as the fusion logic can be pushed down to the Datasets layer.
2. Previously, the separation of `Dataset` and `DatasetPipeline` resulted in `BatchPredictor` needing 2 analgous APIs: `predict` and `predict_pipelined`. This is not needed anymore with streaming as default execution.
3. Previously, the logic for determining batch format was decided in the AIR layer in `Preprocessor` and `BatchPredictor`. This is not needed anymore with zero-copy batch format, as this decision making can be pushed down to the Datasets layer.

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

## API Changes

1. Introduce `map_preprocessor` API directly to Ray Datasets.

```python
def map_preprocessor(preprocessor: Preprocessor, batch_size: Optional[int], prefetch_batches: Optional[int]):
  """Apply the transform of the provided preprocessor to batches of data.

  Args:
    preprocessor: A :class:`~ray.data.preprocessor.Preprocessor` used to transform batches of data. The provided preprocessor is
      expected to already be fitted if it is fittable.
    batch_size: Same as `map_batches`
    prefetch_batches: Same as `map_batches`
  """
```

2. Introduce a `map_predictor` API directly to Ray Datasets.

```python
def map_predictor(predictor: Callable[[], Predictor]], 
                  batch_size: Optional[int], 
                  feature_columns: Optional[List[str]] = None, 
                  keep_columns: Optional[List[str]] = None, 
                  prefetch_batches: Optional[int] = 0, 
                  min_scoring_workers: int = 1,
                  max_scoring_workers: Optional[int] = None,
                  num_cpus_per_worker: Optional[int] = None,
                  num_gpus_per_worker: Optional[int] = None, 
                  **predict_kwargs):
  """Predict on batches of data.

  Args:
    predictor: A function that returns a Predictor to use for inference.
    batch_size: Batch size to use for prediction.
    feature_columns: List of columns in the preprocessed dataset to use for
      prediction. Columns not specified will be dropped
      from `data` before being passed to the predictor.
      If None, use all columns in the preprocessed dataset.
    keep_columns: List of columns in the preprocessed dataset to include
        in the prediction result. This is useful for calculating final
        accuracies/metrics on the result dataset. If None,
        the columns in the output dataset will contain
        just the prediction results.
    min_scoring_workers: Minimum number of scoring actors.
    max_scoring_workers: If set, specify the maximum number of scoring actors.
    num_cpus_per_worker: Number of CPUs to allocate per scoring worker.
      Set to 1 by default.
    num_gpus_per_worker: Number of GPUs to allocate per scoring worker.
      Set to 0 by default. If you want to use GPUs for inference, please
      specify this parameter.
    ray_remote_args: Additional resource requirements to request from ray.
    predict_kwargs: Keyword arguments passed to the predictor's
        ``predict()`` method.
  """
```
      
## Example: Multi-stage Inference Pipeline from pre-trained model

Note that Checkpoints are not on the critical path, but can still be used.

### Before
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

### After
```python
predictor = lambda: TorchPredictor(pretrained_resnet())

# Alternatively, create from AIR Checkpoint.
# predictor = lambda: TorchPredictor.from_checkpoint(air_checkpoint)

predictions = ds.read_parquet("s3://data")
    .map_batches(lambda x: x+1)
    .map_preprocessor(TorchVisionPreprocessor(torchvision.transforms.Crop(224, 224)))
    .map_predictor(predictor)
```

## Example: Implementing a custom Predictor.
Using Checkpoints are no longer necessary.

### Before
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

### After
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

predictor_creator = lambda: MXNetPredictor(gluon.model_zoo.vision.resnet50_v1(pretrained=True))
dataset.map_batches(preprocess, batch_format="numpy").map_predictor(predictor_creator)
```



## Compatibility, Deprecation, and Migration Plan
An important part of the proposal is to explicitly point out any compability implications of the proposed change. If there is any, we should thouroughly discuss a plan to deprecate existing APIs and migration to the new one(s).

Ray 2.4: Onboard new users onto new APIs
- Introduce `map_preprocessor` and `map_predictor` as Alpha APIs.
- Rewrite all batch inference examples to use these APIs, and use `map_batches` directly instead of `BatchMapper` and `Chain`.
- `BatchPredictor` will still be fully supported.

Ray 2.5: Start migrating existing users to new APIs
- Soft Deprecate `BatchPredictor`. Warn if this API is used, and remove from Documentation.
- Add migration guide in documentation for guidance on how to migrate to the new APIs.

Ray 2.6: Deprecate old APIs
- Hard deprecate `BatchPredictor`.


## Test Plan and Acceptance Criteria
The proposal should discuss how the change will be tested **before** it can be merged or enabled. It should also include other acceptance criteria including documentation and examples. 

- Unit and integration for new APIs
- Documentation and examples on new API.
