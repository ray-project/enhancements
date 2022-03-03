# Ray Enhancement Proposals
This repo tracks Ray Enhancement Proposals (REP). The REP process is the main way to propose, discuss, and decide on features and other major changes to the Ray project. 

Each REP should include the following information:
## Summary
### General Motivation
What use cases is this proposal supposed to enhance. If possible, please include details like the environment and scale.
### Should this change be within `ray` or outside?
From a software layering perspective, should this change be part of the main `ray` project, part of an ecosystem project under `ray-project`, or a new ecosystem project?

## Stewardship
### Required Reviewers
The proposal will be open to the public, but please suggest a few experience Ray contributors in this technical domain whose comments will help this proposal. Ideally, the list should include Ray committers. 
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
