# Roll out "strict mode" for Ray Data

## Summary

Make a breaking API change to always require data schemas in Ray Data.

### General Motivation

This REP proposes rolling out a breaking API change to Ray Data, termed "strict mode". In strict mode, support for standalone Python objects is dropped. This means that instead of directly storing, e.g., Python `Tuple[str, int]` instance in Ray Data, users will have to either give each field a name (i.e., `{foo: str, bar: int}`), or use a named object-type field (i.e., `{foo: object}`). In addition, strict mode removes the "default" batch format in place of "numpy" by default. This means that most users just need to be aware of `Dict[str, Any]` (non-batched data records) and `Dict[str, np.ndarray]` (batched data) types when working with Ray Data.

The motivation for this change is to cut down on the number of alternative representations users have to be aware of in Ray Data, which complicate the docs, examples, and add to new user confusion.
For reference, this is the main PR originally introducing strict mode: https://github.com/ray-project/ray/pull/34336

**Full list of changes**
- All read apis return structured data, never standalone Python objects.
- Standalone Python objects are prohibited from being returned from map / map batches.
- Standalone Numpy arrays are prohibited from being returned from map / map batches.
- There is no more special interpretation of single-column schema containing just `__value__` as a column.
- The default batch format is "numpy" instead of "default" (pandas).
- schema() returns a unified Schema class instead of Union[pyarrow.lib.Schema, type].

**Datasource behavior changes**
- `range_tensor`: create "data" col instead of `__value__`
- `from_numpy`/`from_numpy_refs` : create "data" col instead of using `__value__`
- `from_items`: create "item" col instead of using Python objects
- `range`: create "id" column instead of using Python objects

The change itself has been well received in user testing, so the remainder of this REP will focus on the rollout strategy.

### Should this change be within `ray` or outside?
main `ray` project. Changes are made to Ray Data.

## Stewardship
### Required Reviewers
The proposal will be open to the public, but please suggest a few experienced Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.

@amogkam, @c21

### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

@pcmoritz

## Rollout Plan

### Impact of Changes

The proposed change mainly impacts users that are working with in-memory data objects and image datasets. For these users, they will get an error when trying to load data without a schema (e.g., ``StrictModeError: Error validating <data_item>: standalone Python objects are not allowed in strict mode. Please wrap the item in a dictionary like `{data: <data_item>}`. For more details and how to disable strict mode, visit DOC_URL_HERE.``).

### Notification

The main method of notification will be the ``StrictModeError`` exception raised when the user tries to create disallowed data types. The exception will link to documentation on how to upgrade / disable strict mode.

We will also add a warning banner (for a couple releases) on the first import of Ray Data that notifies users of this change.

### Timeline

- Ray 2.5: Enable strict mode by default, with the above notification plan.
- Ray 2.6: No changes.
- Ray 2.7 or after: Enforce strict mode always, and remove code for supporting the legacy code paths.

## Examples:

### Before
```python
ds = ray.data.range(5)
# -> Datastream(num_blocks=1, schema=<class 'int'>)

ds.take()[0]
# -> 0

assert ds.take_batch()
# -> [0, 1, 2, 3, 4]

ds.map_batches(lambda b: b * 2).take_batch()  # b is coerced to pd.DataFrame
# -> pd.DataFrame({"id": [0, 2, 4, 6, 8]})
```

### After
```python
ds = ray.data.range(1)
# -> Datastream(num_blocks=1, schema={id: int64})

ds.take()[0]
# -> {"id": 0}

assert ds.take_batch()
# -> {"id": np.array([0, 1, 2, 3, 4])}

ds.map_batches(lambda b: {"id": b["id"] * 2}).take_batch()  # b is Dict[str, np.ndarray]
# -> {"id": np.array([0, 2, 4, 6, 8])}
```

Note that in the "after" code, the datastream always has a fixed schema, and the batch type is consistently a dict of numpy arrays.

## Test Plan and Acceptance Criteria

The master branch will have strict mode on by default. There will be a suite that tests basic functionality with strict mode off, to avoid regressions.
