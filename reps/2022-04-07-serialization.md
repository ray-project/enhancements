## Summary

### General Motivation

Current Ray's serialization has some issues:

1. Doesn't support out-of-band serialization in Java worker. Cause some performance issues in Java worker.
1. Hard to add a new serializer for a specific class.
1. Doesn't support cross-language serialization for custom classes.

In order to resolve the above issues. We propose to refactor the current serialization code path, provide a standard way for users to develop their own serializers.
In conclusion:

1. Provide pluggable ways for users to implement custom serialization, including:
   1. Cross-language serialization.
   1. Out of band serialization and other optimizations.
2. Unify the current serialization code path to this new pluggable design. Make code cleaner.

### Should this change be within `ray` or outside?

It should be a part of the main `ray` project.

## Stewardship

### Required Reviewers

Hao Chen, Qing Wang, Eric Liang, Zhi Lin, Simon Mo, Jiajun Yao, Siyuan Zhuang

### Shepherd of the Proposal (should be a senior committer)

Hao Chen

## Design and Architecture

### User-level API

#### Register Serializer

Firstly, if users wants to implement custom serialization, they should register their serializer to Ray.
Note that a unique string ID should be provided for class identification in cross-language serialization.

```python
// In Python
ray.register_serializer("ArrowTable", type(arrow_table_obj), ArrowTableSerializer())
ray.register_serializer("Protobuf", type(protobuf_obj), ProtobufSerializer())
```

```java
// In Java
Ray.registerSerializer("ArrowTable", ArrowTable.class, new ArrowTableSerializer());
Ray.registerSerializer("Protobuf", Protobuf.class, new ProtobufSerializer());
```

Ray will maintain a map from ClassID to Serializer in memory. This relationship won't be persisted, which means you need to register them again when the process is restarted.

#### Implement Serializer

Then, the user needs to implement this custom serializer, implements the interface shown below:
Let's take Python as an example. Other languages will be similar.

```python
class MySerializer(RaySerializer):
    def serialize(self, object: MyClass) -> RaySerializationResult:
        pass
    def deserialize(self, in_band_data: bytes,
                    out_of_band_data: List[bytes]) -> MyClass:
        pass

```

`serialize`method should return a `RaySerializationResult`, which contains 2 fields: in-band buffer and out-of-band buffers.
`inBandBuffer` is nothing else than a byte array. Users can use this field to achieve normal in-band serialization.

```python
class RaySerializationResult:
    def __init__(self):
        self.in_band_buffer: bytes = None
        self.out_of_band_buffers: List[RayOutOfBandBUffer] = None
```

`OutOfBandBuffer` is for advanced users. They can use this to do out-of-band optimization. Ray will call `OutOfBandBuffer#write` only once when constructing the final buffer sent to the network.

```python
class MyOutOfBandBuffer(RayOutOfBandBUffer):
    def get_size(self) -> int:
        pass
    # dest is a byte array with size == get_size()
    def write_to(self, dest: bytes):
        pass
```

### Final Buffer Protocol

```
+---------------+----------------+-------------------------------------
|  ClassIDHash  |  in-band data  |   … oob-buf0 … oob-buf1 … oob-buf2 …
+---------------+----------------+-------------------------------------
```

A msgpack buffer which includes 3 parts:
The 1st part is the class ID which is used to find the corresponding serializer.
The 2nd part is the serialized data of the real object.
The last part is a list of out-of-band buffers in memory.

#### Examples

##### Custom Object with OOB Fields

```java
ClassWithOOB {
    int a; // in-band serialized
    byte[] d; // out-of-band serialized
}
```

Here if a user imlements a serializer for this class, he will put `int a` to in-band buffer, and `byte[] d` to the out-of-band buffer.
The buffer layout will be:

| Class ID hash | hash("ClassWithOOB") |
| --- | --- |
| In-band data | a |
| Out of band data list | [d] |

When we get this buffer in another language, we'll firstly check whether a serializer for "ClassWithOOB" is registered. If not, an serialization exception will be thrown. Else, we put in-band and out-of-band data to the `deserialize` method of that serializer, get the result and continue processing.

### Work Items

#### Refactor Serialization Code Path

Unify all existing object serialization to this API.

1. Implement internal object serializer, such as RayError, ActorHandle, etc.
1. By default, if no serializer is registered, use current serializer as the fallback. For example, pickle5 in Python, FST in Java. **In this case, cross-language serialization is disabled.**

#### Implement Individual Popular Formats' Serializers

ArrowTable, Protobuf and so on.

## Compatibility, Deprecation, and Migration Plan

There won't be any user-level compatibility change.

## Test Plan and Acceptance Criteria

Unit tests for core components.
Compatibility is covered by CI.
Additional unit tests for individual serializers in the future.

## Follow-on Work

### Integrate Ray Serializers to Fallback Serializers(To be discussed)

Consider such a class:

```java
class A {
    ArrowTable arrowTable;
}
```

Here in `A`, `ArrowTable`is another class that has registered a serializer to Ray. But `A` hasn't.
Now if we want to serialize `A`, since A hasn't been registered, we will use fallback serializer FST to serialize it. But because FST doesn't know how to serialize `ArrowTable`(only Ray knows), an exception will be thrown.
So firstly we need to discuss whether it's an issue or it's by design.
If it's by design we don't need to do anything. And the compatibility is fine because in the current version it can't be serialized, either. Users need to register class A separately.
If it's an issue and we need to resolve it, a lot of things need to be done. We need to integrate Ray Serializers to FST, Pickle5, and MessagePack.

## Attachments

```python
from typing import List, Set, Dict, Tuple, Optional

class MyClass:
    pass

class RaySerializer:
    def serialize(self, object):
        pass
    def deserialize(self, in_band_data: bytes, out_of_band_data: List[bytes]):
        pass

class RayOutOfBandBUffer:
    def get_size(self) -> int:
        pass
    # dest is a byte array with size == get_size()
    def write_to(self, dest: bytes):
        pass

class MyOutOfBandBuffer(RayOutOfBandBUffer):
    def get_size(self) -> int:
        pass
    # dest is a byte array with size == get_size()
    def write_to(self, dest: bytes):
        pass

class RaySerializationResult:
    def __init__(self):
        self.in_band_buffer: bytes = None
        self.out_of_band_buffers: List[RayOutOfBandBUffer] = None

class MySerializer(RaySerializer):
    def serialize(self, object: MyClass) -> RaySerializationResult:
        pass
    def deserialize(self, in_band_data: bytes,
                    out_of_band_data: List[bytes]) -> MyClass:
        pass

```
