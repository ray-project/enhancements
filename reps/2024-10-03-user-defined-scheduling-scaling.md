# User-defined scheduling and scaling policies for Ray Serve deployments

## Summary
Provide support for user-defined scheduling and scaling policies

## Motivation
### General Motivation
The default scheduling is based on power of 2 choices and the default scaling is based on target ongoing requests/ max ongoing requests. Users of Ray Serve may have different needs such as latency-based SLAs (ie. p99 or p95 requirements) for the requests they serve. This proposal aims to provide a way to configure custom scheduling and scaling policies for Ray Serve deployments with a user-defined policy config, mostly for convenience and out-of-the box support, similar to [multiplexed](https://docs.ray.io/en/latest/serve/model-multiplexing.html) deployments for multi-model serve deployments.

### Should this change be within `ray` or outside?
Inside `ray serve`, as the scheduling and autoscaling policies will be used for out-of-the box deployments. There exist external schedulers such as YuniKorn and Volcano for batch jobs but this is specific to application-level deployments

## Stewardship
### Required Reviewers
The proposal will be open to the public, but please suggest a few experienced Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers.


### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.


## Design and Architecture
The proposal should include sufficient technical details for reviewers to determine the anticipated benefits and risks.

The proposed change requires addition of two parameters called "scaling_policy" and "scheduling_policy" passed to `ray/serve/deployment.py`, such that they can be used in the `ray.serve` decorator, in the form of python `Callable`s.

```python
# Current (pow2 / target_ongoing requests)
@serve.deployment(max_ongoing_requests=1, max_queued_requests=1)
# Proposed
@serve.deployment(scaling_policy=MyAutoscaler, scheduling_policy=MyScheduler)
```

## Compatibility, Deprecation, and Migration Plan
An important part of the proposal is to explicitly point out any compability implications of the proposed change. If there is any, we should thouroughly discuss a plan to deprecate existing APIs and migration to the new one(s).

The following is a backwards-compatible option:
- Check if the old parameters `max_ongoing_requests` and `max_ongoing_requests` are specified without either `scaling_policy` or `scheduling_policy`, then default to initializng the deployment using the default pow2 / target_ongoing requests policies.

- Existing `AutoScalingConfig` class should be refactored to only include the basic parameters across all autoscaling algorithms, which include limits such as min/max replicas. Any reference to `max_queued_requests` `max_ongoing_requests` etc `replica_queue_length_autoscaling_policy`/ target ongoing requests should be moved to its own specific subclass (ie. `RequestLengthPolicy`) or `autoscaling_config`
- The `get_decision_num_replicas` should accept custom arguments as to what is monitored. The `get_policy` already returns a Callable, so it simply needs to be updated to pass variable parameters, depending on which heuristics the user's custom autoscaling policy is required to monitor:

```python
        decision_num_replicas = self._policy(
            curr_target_num_replicas=curr_target_num_replicas,
            total_num_requests=self.get_total_num_requests(),
            num_running_replicas=len(self._running_replicas),
            config=self._config,
            capacity_adjusted_min_replicas=self.get_num_replicas_lower_bound(),
            capacity_adjusted_max_replicas=self.get_num_replicas_upper_bound(),
            policy_state=self._policy_state,
        )
```
to
```python
        decision_num_replicas = self._policy(
            config=self._config,
            policy_state=self._policy_state,
            capacity_adjusted_min_replicas=self.get_num_replicas_lower_bound(),
            capacity_adjusted_max_replicas=self.get_num_replicas_upper_bound(),
            # Autoscaling policy may be based on queue length, SLA violations on each replica, or another heuristic
            **current_metrics,
        )
```

## Test Plan and Acceptance Criteria
The proposal should discuss how the change will be tested **before** it can be merged or enabled. It should also include other acceptance criteria including documentation and examples.

## (Optional) Follow-on Work
Optionally, the proposal should discuss necessary follow-on work after the change is accepted.
