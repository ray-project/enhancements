## Summary

### General Motivation

Current Ray's serialization has some issues:

1. Doesn't support [out-of-band(OOB) data](https://en.wikipedia.org/wiki/Out-of-band_data) in Java workers. So we can't do zero-copy reading/writing. There was a requirement for zero-copy reading Arrow data in Java, but we couldn't achieve it because of this.
2. Type loss in cross-lang serialization for primitive types. e.g. `short` may be deserialized to `int` from a task to another.
3. Doesn't support commonly used container types (e.g. Map).
4. Doesn't support cross-language serialization for custom classes and it's hard to add a new serializer for a specific class.

In order to resolve the above issues. We propose to refactor the current serialization code path, to

1. Provide pluggable ways for users to implement custom serialization, including:
    1. Cross-language serialization.
    2. Out of band serialization and other optimizations.
2. Unify the current serialization code path to this new pluggable design. Make code cleaner. Also, provide a unified interface across different languages.

With a standard way to implement serializers, we can

* Solve issue 4 immediately.
* Solve issue 3 by providing build-in serializers for commonly used classes.
* Solve issue 2 by registering different serializers for primitive types.
* Solve issue 1 by implementing an advanced serializer with OOB optimization.

### Should this change be within `ray` or outside?

It should be a part of the main `ray` project.

## Stewardship

### Required Reviewers

Hao Chen, Qing Wang, Eric Liang, Zhi Lin, Simon Mo, Jiajun Yao, Siyuan Zhuang, Clark Zinzow

### Shepherd of the Proposal (should be a senior committer)

Hao Chen

## Design and Architecture

### User-level API

#### Register Serializer

Firstly, if users want to implement custom serialization, they should register their serializers to Ray.
Note that a unique string ID should be provided for class identification in cross-language serialization.

```python
# In Python
ray.register_serializer("ArrowTable", ArrowTable, ArrowTableSerializer())
ray.register_serializer("MyClass", MyClass, MyClassSerializer())
```

```java
// In Java
Ray.registerSerializer("ArrowTable", ArrowTable.class, new ArrowTableSerializer());
Ray.registerSerializer("MyClass", MyClass.class, new MyClassSerializer());
```

Ray will maintain a map from ClassID to Serializer in memory. This relationship won't be persisted, which means you need to register them again when the process is restarted.

As the first step, this API is only used by Ray core. That means we can simply hard code them without needing persistence.
If we are to expose this API to users, we can consider to add the persistance feature in the future.

We'll also provide a `ray.get_serializer` to get the serializer by either language-specific type or unique string ID, or call `ray.serialize/deserialize` to do serialization directly.

```python
ray.get_serializer("ArrowTable")
ray.get_serializer(ArrowTable)

ray.serialize(arrow_table_obj)
ray.deserialize(ray_serialized_result)
```

#### Implement Serializer

Then, the user needs to implement this custom serializer, implements the interface shown below:
Let's take Python as an example. Other languages will be similar.

```python
class MySerializer(RaySerializer):
    def serialize(self, object: MyClass) -> RaySerializationResult:
        pass
    def deserialize(self, in_band_buffer: bytes,
                    oob_iterator: Iterator[memoryview]) -> MyClass:
        pass

```

`serialize` method should return a `RaySerializationResult`, which mainly contains 2 fields: in-band buffer and out-of-band buffers.

```python
class RaySerializationResult:
    in_band_buffer: bytes
    out_of_band_buffers: Iterable[memoryview]
```

`in_band_buffer` is nothing else than a byte array. When deserializing, in `deserialize` method, Ray will pass `in_band_buffer` back as it as. Users can use this field to achieve normal in-band serialization.

`out_of_band_buffers` is for advanced users, which can be used to achieve zero-copy. When deserializing, Ray will pass an iterator of the OOB buffers to `deserialize` method. Check "Out-of-band serialization" session for examples.

### Example(in-band)

Consider such a class with a transient field. Now the user wants Ray to automatically do the serialization for this class in task arguments and return value when crossing language.

```python
class MyClass:
    state: bytes
    transient_field
```

The only thing the user needs to do is to implement a serializer for this class in corresponding languages and register it:

```python
# In Python
class MyClassSerializer(RaySerializer):

    def serialize(self, instance: MyClass) -> RaySerializationResult:
        return RaySerializationResult(instance.state)

    def deserialize(self, in_band_buffer: bytes,
                    oob_iterator: Iterator[memoryview]) -> MyClass:
        return MyClass(in_band_buffer.obj)

ray.register_serializer("MyClass", MyClass, MyClassSerializer())
```

```java
// In Java
class MyClassSerializer implements RaySerializer {
    @Override
    public RaySerializationResult serialize(MyClass instance) {
        return new RaySerializationResult(instance.state);
    }
    @Override
    public MyClass deserialize(
        ByteBuffer inBandBuffer, Iterator<ByteBuffer> oobIterator) {
        return new MyClass(in_band_data);
    }
}

Ray.registerSerializer("MyClass", MyClass.class, new MyClassSerializer());
```

Now everything is done. Ray will automatically apply your serializer to Ray task's arguments and return value when crossing languages.

### Out-of-band serialization

Let's take `bytearray` as an example.

```python
# In Python
class ByteArraySerializer(RaySerializer):

    def serialize(self, byte_array: bytearray) -> RaySerializationResult:
        return RaySerializationResult(None, [memoryview(byte_array)])

    def deserialize(self, in_band_buffer: bytes,
                    oob_iterator: Iterator[memoryview]) -> bytearray:
        return next(oob_iterator).obj
```

Now `bytearray` objects will be out-of-band serialized.

In the same process, no copy will happen to out-of-band buffers.

For example, in the following code, no copy will happen:

```python
b = bytearray(b'\x01\x02\x03')
serialization_result: RaySerializationResult = ray.serialize(b)
deserialized_bytes: bytearray = ray.deserialize(serialization_result)
```

`deserialized_bytes` and `b` will be backed by the same memory.

When the serialized result needs to go through network, for example, in cross-language situation, only 1 copy will happen. That is from `RaySerializationResult` to the network buffer.

### Nested Serialization

Let's take `List[bytearray]` as an example to see how to handle nested types. We've already implemented a serializer for `bytearray` above.

Note that in real world we might simply wrap Msgpack to do serialization for common classes. Here we rebuild the wheel only to show how it works.

```python
class ListSerializer(RaySerializer):

    def serialize(self, list_object: List[bytearray]) -> RaySerializationResult:
        result = RaySerializationResult()
        result.in_band_buffer = bytearray()
        # Total element count.
        result.in_band_buffer.extend(to_bytes(len(list_object))))
        for element in list_object:
            element_res = ray.serialize(element)
            # in band buffer length
            result.in_band_buffer.extend(to_bytes(len(element_res.in_band_buffer)))
            # in band buffer
            result.in_band_buffer.extend(element_res.in_band_buffer)
            # append out of band buffer
            result.out_of_band_buffers += element_res.out_of_band_buffers
        return result

    def deserialize(self, in_band_buffer: bytes,
                    oob_iterator: Iterator[memoryview]) -> List[bytearray]:
        result = []
        pos = 0
        # read element count
        count, pos = get_int(in_band_buffer, pos)
        for i in range(count):
            # read element size
            element_size, pos = get_int(in_band_buffer, pos)
            # read element in-band data
            element_in_band_data = in_band_buffer[pos: pos + element_size]
            pos = pos + element_size
            # nested serialization here
            element = ray.deserialize(element_in_band_data, oob_iterator)
            result.append(element)
        return result
```

### Fallback Serializer

By default, if no serializer is registered, use serializer in current Ray's version as the fallback. For example, pickle5 in Python, FST in Java.

We'll register a fallback serializer to Ray. For example, pickle5 fallback serializer will be something like the following, all OOB buffers of pickle5 will be wrapped to Ray's OOB buffer.

```python
# Pseudo code
class Pickle5FallbackSerializer(RaySerializer):
    def serialize(self, instance) -> RaySerializationResult:
        pickle_oob_buffers = []
        in_band = pickle5.dumps(instance, buffer_callback=pickle_oob_buffers.append)
        ray_oob_buffers = convert_to_ray_oob_buffers(pickle_oob_buffers)
        return RaySerializationResult(in_band, ray_oob_buffers)

    def deserialize(self, in_band_buffer: bytes,
                    oob_iterator: Iterator[memoryview]):
        return pickle5.loads(in_band_buffer, buffers=list(oob_iterator))
```

Nested serialization will still work in this case.
When registering fallback serializer, we can provide a `register_callback` to receive notifications of serializer registration.

```python
def register_fallback_serializer(serializer, register_callback=None):
    pass

def register_serializer(class_name: str, class_type: type, serializer):
    # Register to Ray
    ...
    # Notify fallback serializer
    __fallback_serializer_notify_fn(class_name, class_type, serializer)
```

In this callback we'll register Ray serializer to the fallback serializer(with some wrapping).

```python
# Pseudo code
def register_to_pickle5(class_name: str, class_type: type, serializer):
    # Wrap Ray's serializer
    def _serialize_wrapper(obj):
        def _deserialize_wrapper(in_band, pickle_oob_buffers):
            ray_oob_buffers = convert_to_ray_oob_buffer(pickle_oob_buffers)
            return serializer.deserialize(in_band, iter(ray_oob_buffers))

        ray_result = serializer.serialize(obj)
        pickle_oob_buffers = convert_to_pickle_oob_buffers(ray_result.out_of_band_buffers)
        return (
            _deserialize_wrapper,
            (ray_result.in_band_buffer, pickle_oob_buffers)
        )
    # Register to pickle5
    pickle.CloudPickler.dispatch[class_type] = _serialize_wrapper

ray.register_fallback_serializer(Pickle5FallbackSerializer(), register_to_pickle5)
```

With this, we can make full use of the sophisticated serialization library. In one language, Users don't need to write boring serializers for every custom class. They only need to write serializers for some key classes.

**Cross-language serialization is disabled in this case.** since the protocol are different between different frameworks.
If users want to do cross-language serialization, unfortunately, they still need to implement a simple wrapper serializer.

This can be mitigated if we can find a full-featured cross-language serialization library as the fallback serializer. But we can't find one by now. Maybe we can use [Fury](https://docs.google.com/document/d/1nrKrXnyRqiIQqLV1P6i3t6TXQEwduoyLeHLMT2DV-fc/edit?usp=sharing) in the future after it's open-source.

### Final Buffer Protocol

```plaintext
+---------------+----------------+------------------------
|  ClassIDHash  |  in-band data  |  oob-buf1, oob-buf2...
+---------------+----------------+------------------------
```

A msgpack buffer which includes 3 parts:
The 1st part is the class ID which is used to find the corresponding serializer.
The 2nd part is the serialized data of the real object.
The last part is a list of out-of-band buffers in memory.

### Work Items

* [P0] Refactor Serialization Code Path, unify all existing serialization code to this API.
* [P1] Serializers for common container classes like list and dict.
* [P2] Serializers for Popular Formats, e.g. Arrow.

## Compatibility, Deprecation, and Migration Plan

There won't be any user-level compatibility change.

## Test Plan and Acceptance Criteria

* Unit tests for core components.
* Compatibility is covered by CI.
* Performance benchmarks.
* Additional unit tests for individual serializers in the future.
