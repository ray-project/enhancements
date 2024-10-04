# User-defined scheduling and scaling policies for Ray Serve deployments

## Summary
Provide support for user-defined scheduling and scaling policies

## Motivation

### General Motivation

The default scheduling is based on power of 2 choices and the default scaling is based on target ongoing requests/ max ongoing requests. Users of Ray Serve may have different needs such as SLAs for the requests they serve. This proposal aims to provide a way to configure custom scheduling and scaling policies for Ray Serve deployments, with a user-defined policy config.


=============== 

What use cases is this proposal supposed to enhance. If possible, please include details like the environment and scale.
### Should this change be within `ray` or outside?
From a software layering perspective, should this change be part of the main `ray` project, part of an ecosystem project under `ray-project`, or a new ecosystem project?

When reviewing the REP, the reviewers and the shepherd should apply the following judgements:
- If an author proposes a change to be within the `ray` repo, the reviewers and the shepherd should assess whether the change can be layered on top of `ray` instead. 
If so we should try to make the change in a separate repo. 
- For a change proposed as an ecosystem project under `ray-project`: the reviewers and the shepherd should make sure that the technical quality
meets the bar of (at least) a good "experimental" or "alpha" feature -- we should be comfortable welcoming Ray users with similar use cases to try this project.
- For a change proposed as a new ecosystem project (outside of `ray-project`): then this REP is just serving as a "request for comments". 
We don't need to go through the voting process, since it's not Ray committers' decision to approve the change. 

## Stewardship
### Required Reviewers
The proposal will be open to the public, but please suggest a few experienced Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers. 
### Shepherd of the Proposal (should be a senior committer)
To make the review process more productive, the owner of each proposal should identify a **shepherd** (should be a senior Ray committer). The shepherd is responsible for working with the owner and making sure the proposal is in good shape (with necessary information) before marking it as ready for broader review.

## Design and Architecture
The proposal should include sufficient technical details for reviewers to determine the anticipated benefits and risks.

## Compatibility, Deprecation, and Migration Plan
An important part of the proposal is to explicitly point out any compability implications of the proposed change. If there is any, we should thouroughly discuss a plan to deprecate existing APIs and migration to the new one(s).

## Test Plan and Acceptance Criteria
The proposal should discuss how the change will be tested **before** it can be merged or enabled. It should also include other acceptance criteria including documentation and examples. 

## (Optional) Follow-on Work
Optionally, the proposal should discuss necessary follow-on work after the change is accepted.
