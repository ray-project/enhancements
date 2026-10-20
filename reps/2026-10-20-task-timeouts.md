## Summary
### General Motivation

Ray has no way to bound how long a single task or actor method call runs. today users either pass `timeout` to `ray.get` (which only stops the caller from waiting and leaves the task running and holding resources), or write watchdog code around `ray.cancel`. this has been requested for years in [ray-project/ray#18916](https://github.com/ray-project/ray/issues/18916), and a maintainer asked for a short REP before any code.

this REP proposes a `timeout_s` option on remote functions, actor classes (as a default for methods), and actor methods. when the deadline passes, the owner fails the task's return refs with a new `TaskTimeoutError`. for normal tasks the owner also cancels the running attempt on a best-effort basis. for actor tasks only the ref fails and the actor is left alone.

common use cases:
- batch and data pipelines where one stuck input (bad file, hung network call) blocks a whole job
- serving or agent style workloads that call tools or models through actors and need a hard per-call budget
- fault tolerance: a hung attempt can be retried instead of stalling forever

### Should this change be within `ray` or outside?

within ray. the deadline has to live with the owner of the task (the core worker `TaskManager`), it has to interact with retries, cancellation, lineage reconstruction and streaming generators, and it has to produce a typed error stored into the return objects. none of this can be done correctly from a library on top of ray.

## Stewardship
### Required Reviewers
@jjyao @edoakes (to be confirmed by the shepherd)

### Shepherd of the Proposal (should be a senior committer)
@jjyao (proposed)

## Design and Architecture

### Current state (for reference)

- options are validated in `python/ray/_common/ray_option_utils.py` (`_common_options`, `_task_only_options`, `_actor_only_options`), then passed from `RemoteFunction._remote` (`python/ray/remote_function.py`) to `CoreWorker.submit_task`, and from `ActorMethod._remote` / `ActorClass._remote` (`python/ray/actor.py`) to `CoreWorker.submit_actor_task` / `create_actor` in `python/ray/_raylet.pyx`.
- `_raylet.pyx` builds a C++ `TaskOptions` (`src/ray/core_worker/common.h`), which `CoreWorker::SubmitTask` / `SubmitActorTask` turn into a `TaskSpecification` (`TaskSpec` in `src/ray/protobuf/common.proto`).
- the owner tracks every submitted task in `TaskManager` (`src/ray/core_worker/task_manager.cc`): `AddPendingTask`, `CompletePendingTask`, `RetryTaskIfPossible`, `FailOrRetryPendingTask`, `FailPendingTask`, `MarkTaskCanceled`, and `ResubmitTask` for lineage reconstruction.
- cancellation goes through `CoreWorker::CancelTask`, which routes to `NormalTaskSubmitter::CancelTask(task_spec, force_kill, recursive)` or `ActorTaskSubmitter::CancelTask(task_spec, recursive)` in `src/ray/core_worker/task_submission/`. the executor side is `CoreWorker::HandleCancelTask`.
- `ray.get(timeout=...)` raises `GetTimeoutError(RayError, TimeoutError)` from `python/ray/exceptions.py`; cancellation surfaces as `TaskCancelledError` / `ErrorType.TASK_CANCELLED`.

### API

```python
@ray.remote(timeout_s=30)
def f(x): ...

f.options(timeout_s=5).remote(x)

@ray.remote(timeout_s=10)          # default for every method of this actor
class A:
    @ray.method(timeout_s=2)       # per method default
    def g(self): ...

a = A.remote()
a.g.options(timeout_s=1).remote()  # per call override
```

- `timeout_s`: positive float, or `None` (default, no timeout). validated in `ray_option_utils.py`; added to `_common_options` for tasks and methods. on actor classes it is the default for method calls and does not apply to actor creation (see open questions).
- precedence for actor calls: call `.options()` > `@ray.method` > actor class default.
- new exception in `python/ray/exceptions.py`:

```python
class TaskTimeoutError(GetTimeoutError):
    """the task or actor call did not finish within timeout_s."""
    # fields: task_id, function_name, timeout_s, attempt_number
```

subclassing `GetTimeoutError` (and therefore `TimeoutError`) means existing `except GetTimeoutError` / `except TimeoutError` code around `ray.get` also handles the new case.

### Plumbing

1. python: `timeout_s` flows through `RemoteFunction._remote` and `ActorMethod._remote` into `submit_task` / `submit_actor_task` in `_raylet.pyx`.
2. `TaskOptions` gets `std::optional<double> timeout_s` (or `int64_t timeout_ms`, -1 for unset).
3. `TaskSpec` proto gets a new field `int64 timeout_ms`, set in `TaskSpecBuilder`. it is carried in the spec so the executor and the dashboard / task events can see it.
4. new `ErrorType` value `TASK_TIMEOUT = 35` in `common.proto`, mapped to `TaskTimeoutError` in the python error deserialization path (the same place `TASK_CANCELLED` maps to `TaskCancelledError`).

### Deadline tracking

- deadlines live in the owner's `TaskManager`, not in the executor. the owner is the only process that always survives a worker hang or crash and already owns retry decisions.
- the clock starts per attempt when the task is dispatched to a worker (lease granted and `PushTask` sent), not at `.remote()`. time spent waiting for dependencies or resources does not count. reason: users want to bound execution, and counting queue time would make timeouts fire under normal cluster pressure. a separate end to end deadline is an open question.
- implementation: `TaskManager` keeps a min-heap (or ordered map) of `(deadline, task_id, attempt_number)` entries, drained by one `boost::asio` timer or the existing `PeriodicalRunner` at a small interval (config `task_timeout_check_interval_ms`, default 100ms). entries are inserted when the submitter reports dispatch and lazily skipped if the task already finished or moved to a new attempt. no per-task timers, so cost is O(log n) per timed task and zero for tasks without `timeout_s`.
- on expiry, `TaskManager` calls a new `HandleTaskTimeout(task_id, attempt_number)` that:
  1. checks the attempt is still the current running attempt (guards against races with `CompletePendingTask`)
  2. triggers cancellation of the attempt (below)
  3. calls `FailOrRetryPendingTask(task_id, rpc::ErrorType::TASK_TIMEOUT, ...)`, so the normal retry path decides whether to resubmit or to store `TaskTimeoutError` into the return objects.
- once the error is stored, a late reply from the worker is ignored, the same way `CompletePendingTask` already ignores replies for tasks that are no longer pending or are on an older attempt.

### Cancellation on timeout

- normal tasks: best-effort cancel through `NormalTaskSubmitter::CancelTask(spec, force_kill=false, recursive=true)`. if the task is still queued it is removed, if it is running the executor gets the usual `KeyboardInterrupt` path from `HandleCancelTask`. a follow-up `force_kill=true` after a grace period is an open question; by default we do not kill the worker because the user did not ask for that. the ref fails immediately on expiry, it does not wait for the cancel to be acknowledged.
- actor tasks: only the ref fails. we do not call `ActorTaskSubmitter::CancelTask` by default and never kill the actor, because actor methods share state and interrupting them can corrupt it. for async actors an opt-in to cancel the coroutine is an open question. the actor keeps running the method and its result is dropped.
- children: `recursive=true` matches `ray.cancel` defaults for normal tasks. actor tasks spawn nothing on our behalf since we do not cancel them.

### Retries

- the timeout is per attempt. each retry gets a fresh `timeout_s`.
- timeouts are not retried by default. `TASK_TIMEOUT` is treated like an application-level failure: it is retried only when `retry_exceptions=True` or `TaskTimeoutError` is listed in `retry_exceptions`, and it counts against `max_retries` (tasks) or `max_task_retries` (actor methods). system failures that happen before the deadline (worker died, node died) keep today's behavior.
- reason: a task that hung once will often hang again, and silent retries could multiply runtime by `max_retries` (default 3) without the user noticing.

### Generators and streaming

- for `num_returns="streaming"` / `"dynamic"`, the deadline covers the whole generator attempt, not each item. items already reported through `HandleReportGeneratorItemReturns` stay readable; on expiry the stream ends with `TaskTimeoutError` as the next item, same as any other generator failure.
- a per-item timeout (`timeout_s_per_item`) is left for later (open question). backpressure waits (`generator_backpressure_num_objects`) do count toward the deadline because the task is still holding a worker; this is documented.

### Lineage reconstruction

- `ResubmitTask` reuses the original spec, so the same `timeout_s` applies to each reconstruction attempt with the same per-attempt rule.
- if reconstruction times out and has no retries left, the object fails with `TaskTimeoutError` wrapped the same way `OBJECT_UNRECONSTRUCTABLE_*` errors are today. we may add `OBJECT_UNRECONSTRUCTABLE_TASK_TIMEOUT` for clarity (open question).

### Interaction with `ray.get(timeout=...)`

- independent. `ray.get(timeout)` bounds how long the caller waits and leaves the task alone; `timeout_s` bounds the task and fails the ref for every reader.
- if both are set and `ray.get`'s timeout fires first, the caller gets plain `GetTimeoutError` and the task keeps running. if `timeout_s` fires first, the caller gets `TaskTimeoutError`. code can tell them apart with `isinstance(e, TaskTimeoutError)`.
- same for `ray.wait`: a timed out ref counts as ready (it holds an error).

### Observability

- task events record the timeout as a failure with `error_type=TASK_TIMEOUT`, visible in the dashboard and `ray list tasks`.
- a counter metric for timed out tasks, tagged by task type.

### Compatibility

- purely additive. `timeout_s=None` is the default and tasks without it take no new code path beyond a null check.
- new proto field and enum value are backward compatible; old workers ignore the field because deadlines are enforced by the owner.
- `TaskTimeoutError` subclasses `GetTimeoutError`, so existing handlers keep working.
- C++ and Java APIs can expose the option later through the same `TaskOptions` field.

## Compatibility, Deprecation, and Migration Plan

no deprecation. users with watchdog code around `ray.cancel` can migrate to `timeout_s`. documentation gets a section in the tasks and actors fault tolerance guides.

## Test Plan and Acceptance Criteria

unit (C++, `src/ray/core_worker/tests/`):
- `task_manager_test.cc`: deadline fires and stores `TASK_TIMEOUT`; completion just before the deadline wins; a late reply after timeout is ignored; stale attempt entries are skipped; retry with `retry_exceptions` resets the deadline and counts against `max_retries`.
- `normal_task_submitter_test.cc`: timeout calls `CancelTask` for queued and running tasks.
- `actor_task_submitter_test.cc`: timeout fails the ref without sending a cancel or killing the actor; later calls on the actor still succeed.

python (`python/ray/tests/`):
- option validation (negative, zero, non-numeric, `None`) and precedence for actor class, `@ray.method`, and `.options()`.
- `ray.get` raises `TaskTimeoutError`, which is caught by `except GetTimeoutError` and `except TimeoutError`.
- task sleeping past the deadline is interrupted and its worker is reused.
- `retry_exceptions=[TaskTimeoutError]` retries with a fresh deadline; default does not retry.
- streaming generator: items before the deadline are readable, then `TaskTimeoutError`.
- lineage reconstruction of a timed task respects `timeout_s`.
- queue time does not count: task blocked on resources does not time out.
- no regression in microbenchmarks for tasks without `timeout_s`.

acceptance: all of the above pass in CI, plus docs and an example.

## (Optional) Follow-on Work

- end to end deadline including queue time (`deadline_s`)
- per-item timeout for streaming generators
- opt-in cancel for async actor coroutines on timeout
- escalation to `force_kill` after a grace period
- C++ and Java API support

## Open Questions

1. should the clock start at dispatch (proposed) or at `.remote()`? or offer both?
2. should `timeout_s` on an actor class also bound actor creation (`__init__`), or only method calls?
3. should normal task timeouts escalate to `force_kill` after a grace period, and should that be configurable?
4. for async actors, should we cancel the coroutine on timeout by default, since that is safer than interrupting threaded actors?
5. should timeouts be retried by default like system failures, or only when opted in (proposed)?
6. do we want a dedicated `OBJECT_UNRECONSTRUCTABLE_TASK_TIMEOUT` error type for reconstruction failures?
7. check interval: is 100ms resolution acceptable, or should we use an exact asio timer for the nearest deadline?
