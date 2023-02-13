## Summary

### General Motivation

Currently, the supported workers in Ray are Python, Java, and C++. However, there are many other languages that are popular and have a large user base. For example, JavaScript, Go, and Rust, etc. It would be great if we can support these languages as well. However, adding support for these languages is not trivial. First of all, the language specific workers need to be supported in Ray core. Secondly, the relevant language specific libraries need to be developed such as Ray AIR.

To make it easier to support new languages, we can use WebAssembly (WASM) to support new languages. WASM is a portable binary instruction format for a stack-based virtual machine. It is designed as a compilation target and can run with a minimum amount of overhead. There is an increasing number of languages that can be compiled to WASM, such as C/C++, Rust, Go. In addition, with the embedded WASM interpreter, we can support languages that cannot be compiled to WASM, such as JavaScript, Python, etc.

Furthermore, it is also possible that we enable transparent Ray support in WASM. As a result, many legacy applications can be easily migrated to Ray and run in a distributed manner and Ray layer can be transparent to the application. This will be a huge benefit for Ray adoption in the generic serverless computing environment.

In our internal FaaS environment, we have WASM function support. These functions are compiled from different programming languages such as Javascript, Rust, Go, C/C++, etc. Compared with normal container based FaaS solutions such as Knative, WASM based functions can be loaded with a shorter period of time while still providing the same or even better secure running environments. So far, WASM functions are still far from perfect. There are few pain points for WASM functions.

1. If different functions want to communicate with each other, they need use either RPC, message queue or KV store. However, these solutions are not convenient for the developers. The FaaS function developers need a simpler communication mechanism between different functions and run tasks in parallel.
2. WASM function is executed in a single thread. Although multi-threading support is on the way, it is still not mature enough. To run tasks in a single FaaS function in parallel, running WASM with Ray API support is a good solution. Besides, running a WASM function distributedly in Ray has the FT support out of the box.
3. Our FaaS environment has a large number of edge servers. The resources on each of the servers are limited. It is not possible to provide more resources for a single function than the resources on the server. In Ray, all the resources in the Ray cluster is treated as a single resource pool. If we can support Ray in our FaaS environment, we can run WASM functions with larger resource requirements.

As a result, Having Ray support in our FaaS WASM environment will be a huge benefit for us and it is a very important feature for us.

One use case of WASM on Ray is the ML inference tasks. In our enterprise environment, we need to support running ML inference tasks in Ray and make these ML inference tasks share the Ray cluster resources. Compared to the current Ray Serve ML inference solution, which needs to keep the ML inference tasks running in the Ray cluster memory, the WASM ML inference solution can reduce the resource consumption of the Ray cluster. Currently, WASM runtime such as WasmEdge already support running ML inference tasks through WASI-NN support.

In this REP, we will discuss the design of WASM support in Ray. We will first discuss the general architecture of WASM support in Ray. Then we will discuss how to support Ray APIs in WASM. Finally, we will discuss an example of how to use Ray APIs in WASM.

### Should this change be within `ray` or outside?

This change should be within `ray` repo. The WASM support in Ray is a core feature of Ray. It is not possible to implement it in a separate repo.

## Stewardship

### Required Reviewers

@iycheng, @scv119, @ericl

### Shepherd of the Proposal (should be a senior committer)

@ericl

## Design and Architecture

### General Architecture

The WASM support in Ray can be generally divided into two parts: the WASM runtime and the WASM worker.The WASM runtime is the core part of WASM support in Ray. It is also compiled into the WASM worker.

The WASM runtime is responsible for loading and executing WASM binaries. Furthermore, it exports Ray API access to the loaded WASM binaries through WASM host calls.

The design of WASM worker is similar to the design of existing workers in Ray. It registers to the `raylet` process and handles the task execution requests from the `raylet` process. The difference is that the WASM worker loads and executes the WASM binary by taking advantage of the WASM runtime.

Here, we demonstrate the general design architecture of WASM support in Ray. It is shown in the following figure. This diagram is only showing the relationship between the components. In reality, WASM runtime, WASM worker, and Ray core are compiled into a single binary.

```
+-------+ +-------+ +-------+
|       | |       | |       |
| WASM  | | WASM  | | WASM  |
| Task  | | Task  | | Task  |
|       | |       | |       |
+-------+ +-------+ +-------+
+---------------------------+
|                           |
|       WASM Runtime        |
|                           |
+---------------------------+
+---------------------------+
|                           |
|        WASM Worker        |
|                           |
+---------------------------+
+---------------------------+
|                           |
|         Ray Core          |
|                           |
+---------------------------+

```

### Components

To support executing WASM binaries in Ray, we need several different components so that the user can easily use them. For example, we need a tool to load the compiled WASM binary and launch distributed tasks. We need a WASM worker executable for Ray to launch the remote WASM tasks.

The WASM support in Ray consists of the following components:

1. `raywa`: command line tool for loading and executing WASM binaries in a Ray cluster.
2. `wasm_worker`: the WASM worker that loads and executes WASM binaries.
3. `libray_wasm_api.so`: the WASM runtime used by the `wasm_worker` and `raywa`.

## Usage Example

Here we will show how to use WASM support in Ray. This will be common for all languages that can be compiled to WASM. For example, C/C++, Rust, Go, etc. But in this example, we will use C as an example.

### Import Ray API in WASM

Here is an example C code that uses Ray API.

```c
#include "stdio.h"

__attribute__((import_module("ray"), import_name("call"))) uint32_t rcall(void *, ...);

// macro to call remote function
#define REMOTE(p, ...) rcall((void *)(p), ##__VA_ARGS__)

int add(int a, int b) {
  return a + b;
}

int main() {
  REMOTE(add, 122, 233);
  return 0;
}
```

The `__atribute__((import_module("ray"), import_name("call")))` is used to import the Ray API to call a remote function. The `rcall` function is the new name of the imported Ray API. The `REMOTE` macro is used to call the remote function. The first argument of the `REMOTE` macro is the function pointer. The following arguments are the arguments of the remote function.

### Compile C code to WASM

To compile C/C++ code to WASM, we need to use the `clang` compiler. The `clang` compiler supports compiling C/C++ code to WASM. Here is the command to compile the above C code to WASM.

```bash
/<path_to_wasi_sdk>/bin/clang --sysroot /<path_to_wasi_sdk>/share/wasi-sysroot/ -Wl,--export-all <my_file>.c -o <my_file>.wasm

```

Here, we use `--export-all` to export all the symbols in the WASM binary. This is because the WASM worker needs to access the called function in the WASM binary when executing the remote function. Finally, we get the generated WASM binary.

### Other Languages Support

#### Rust
It is also possible to support other languages that can be compiled to WASM. For example in Rust, you can declare the Ray APIs you want to import in the following way.

```rust
#[link(wasm_import_module = "ray")] 
extern "C" {
  #[link_name="call"]
  fn rcall(ptr: *const u8, ...) -> u32;
}

```

Then you can use the `rcall` function to call the remote function. The `ptr` is the function pointer. The following arguments are the arguments of the remote function.


#### Other Languages

To use Ray APIs in JavaScript, the Ray APIs need to be exported to the Javascript runtime environment. For example, if the user is using SpiderMonkey, the Ray APIs can be exported to SpiderMonkey and then used in JavaScript.

Futhermore, the new Component Model standard is supposed to be ready very soon. It will provide a standard way to import and export components. We can use the new Component Model standard to declare Ray APIs in WASM so that it will be easier to use Ray APIs in different languages.


## Pros and Cons
Compared with directly supporting other languages, WASM support in Ray has the following pros and cons.

### Pros:
1. It is easy to support new languages. For example, if we want to support a new language, we only need to compile the language to WASM and then use the WASM support in Ray to run the WASM binary.
2. Offer the ability to run Ray-As-A-Service. With WASM enabled in Ray, it is possible for users to use untrusted third-party WASM binaries in Ray. As WASM supports a sandboxed environment, the activities of the untrusted third-party WASM binaries can do is limited.
3. Lightweight Workers. WASM is only 10% the size of Python for comparable tasks. This means that the WASM worker is much smaller than the Python worker. WASM is lightweight and fast to load. It is possible to load the WASM binary in a few milliseconds. Compared to Python, it is much faster to load the WASM binary.
4. Architecture Agnostic. WASM provides an abstract layer on top of the hardware. It is possible to support different hardware architectures in the same WASM binary.
5. Easy to support different versions of the same language. For example, if we want to support different versions of JavaScript, we only need to compile different versions of JavaScript to WASM and then use the WASM support in Ray to run the WASM binary.

### Cons:
1. It is not easy to support all Ray APIs in WASM. For example, there is a concept of `Actor` in Ray. `Actor` is a special type of task that can maintain state. However, WASM is a collection of functions, a stack, and a heap. It is not a simple task to map the concept of `Actor` to WASM. Therefore, we will not support `Actor` in WASM in the first version of WASM support in Ray.
2. WASM is a new technology. It is not mature enough. For example, the WASI standard is still under development.
3. WASM is not as fast as native code. However, it is possible to optimize WASM code to make it as fast as native code. For example, LLVM provides a tool called `wasm-opt` to optimize WASM code. We can use `wasm-opt` to optimize the WASM code generated by the `clang` compiler.
4. Lack of support for some libraries. For example, to do ML in WASM, we need to support some ML libraries such as Apache Arrow, TensorFlow, etc. However, there are not many ML libraries that support WASM. We need to find a way to support these libraries in WASM.


Futhermore, Ray support in WASM can be transparent to the user. For example, the user can write a code in Rust and then compile to WASM. With some advanced features implemented, it is possible to automatically run the WASM binary distributedly in Ray without the user knowing his/her code is running in Ray.


## Proposed Ray APIs in WASM

Ideally, we want to support all Ray APIs in WASM. However, there are some limitations in the current WASM design. For example, there is a concept of `Actor` in Ray. `Actor` is a special type of task that can maintain state. However, WASM is a collection of functions, a stack, and a heap. It is not a simple task to map the concept of `Actor` to WASM. Therefore, we will not support `Actor` in WASM in the first version of WASM support in Ray.

Here is the list of Ray APIs that we will support in WASM in the first version. In the future, we will add more Ray APIs to support more use cases.

```c

typedef struct {
  uint16_t cap;
  uint16_t len;
  uint8_t data[0];
} ObjectRef;

/**
 * Call a remote function. This is the proposed version of Ray remote function call API. It is different from previous POC version.
 *
 * @param ptr The pointer to the  the function.
 * @param object_ref The ObjectRef object allocated by the caller to store the return value.
 * @param ... The arguments of the remote function.
 * @return The ObjectRef object. (NULL if failed or the given object_ref is not valid.)
 */
__attribute__((import_module("ray"), import_name("call"))) ObjectRef *rcall(void *, ObjectRef *, ...);

/**
 * Get an object from the object store.
 *
 * @param object_ref The object reference of the object to get.
 * @param ptr The pointer to the memory location to store the object.
 * @param size The size of the allocated memory for the return object.
 * @param size_out The size of the returned object.
 * @return The pointer to the memory location of the object. (NULL if failed.)
 */
__attribute__((import_module("ray"), import_name("get"))) void *rget(ObjectRef *, void *, size_t, size_t *);


/**
 * Wait for a list of objects to appear in the object store.
 *
 * @param object_refs The list of object references to wait for.
 * @param ready_list The list of object references index that are ready.
 * @param size The size of the allocated ready_list.
 * @param num_objects The number of objects to wait for.
 * @param num_returns The number of objects to return.
 * @param timeout_ms The timeout in milliseconds.
 * @return The list of object references returned. (NULL if failed.)
 */
__attribute__((import_module("ray"), import_name("wait"))) uint32_t rwait(ObjectRef **, uint32_t *, size_t, uint32_t, uint32_t, uint64_t);

/**
 * Put an object into the object store.
 *
 * @param ptr The pointer to the memory location of the object.
 * @param size The size of the object.
 * @param object_ref The ObjectRef object allocated by the caller to store the object reference.
 * @return The ObjectRef object. (NULL if failed or the given object_ref is not valid.)
 */
__attribute__((import_module("ray"), import_name("put"))) ObjectRef *rput(void *, uint32_t, ObjectRef *);

/**
 * Cancel a task.
 *
 * @param object_ref The object reference of the task to cancel.
 * @param force_kill Whether to force kill the task.
 * @return Whether the task is canceled.
 */
__attribute__((import_module("ray"), import_name("cancel"))) int rcancel(ObjectRef *, bool);

```

## MVP Feature List

The MVP feature list is as follows:
* Support Command Line Interface (CLI) to load WASM binary and start Ray WASM driver task.
* Support Ray task calls in WASM.
  * Support Basic WASM supported types as arguments and return values.
* Support Ray cancel task in WASM.
* Support Ray object set/get in WASM.
* Support Ray wait for objects in WASM.


## Compatibility, Deprecation, and Migration Plan

## Test Plan and Acceptance Criteria

To make sure the WASM support in Ray works, we need to add the following tests:
* Make sure the Ray APIs in WASM works for basic use cases.
* Make sure the Ray APIs calls do not cause crashes or memory leaks.
* Make sure the inter-language calls either work or fail gracefully.
* Make sure the Ray WASM worker can be started and stopped correctly with different configurations.

We will also need to test FT by randomly killing Ray WASM workers and make sure the Ray cluster can still work correctly and the tasks can be rescheduled to other Ray WASM workers.

Acceptance Criteria:
With the above tests, we can make sure the Ray WASM support works correctly.


## (Optional) Follow-on Work

With MVP feature list implemented, we can continue to add more Ray APIs to support more use cases. For example:
* Add support for `Actor`. 
* Add support for more data types. 
* Advanced JavaScript support.
* More Ray core APIs support.
