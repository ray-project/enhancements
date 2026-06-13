# Question / feature request: Is there an MCP server for managing Ray on Kubernetes — and if not, why not?

_**Status:** RFC / request for comments. Per the REP process, a potential **new
ecosystem project outside `ray-project`** is filed as an RFC and requires no vote.
This is not a formal REP proposing a specific implementation — it's a question
about why an agent-facing control surface for Ray-on-Kubernetes doesn't appear to
exist yet, filed here to reach maintainers who'd know the history. If a lighter
venue (Discussion/Slack) is preferred, happy to move it._

---

## The question

Is there an existing or planned **Model Context Protocol (MCP) server** for
managing Ray on Kubernetes (KubeRay) — i.e. a server an AI agent could connect to
in order to create/inspect `RayCluster`s, submit and follow `RayJob`s, and manage
`RayService`s, with Ray-aware semantics?

I went looking and couldn't find one that fits. Before assuming there's a gap, I
wanted to ask the people who'd know: **is this missing on purpose?** Has it been
considered and rejected, is it on a roadmap somewhere, or has it just not come up?

## What I found while looking (please correct anything wrong here)

- The **`ray-kubectl-plugin`** REP describes a great developer experience — but
  it's explicitly a **human-facing CLI** ("for data scientists and AI researchers
  unfamiliar with Kubernetes"). It doesn't seem aimed at an *agent* driving Ray
  programmatically, and its dashboard story is "open the browser UI."
- The **`kuberay-authentication`** REP confirms the Ray dashboard is unauthenticated
  by default and proposes TokenReview-based auth — relevant to any tool that would
  talk to the dashboard/job API.
- Generic Kubernetes MCP servers exist and *can* CRUD the KubeRay CRDs by raw
  `apiVersion`+`kind`, but they have no Ray awareness — they can't, for example,
  tell an agent *why* a job is `Pending` or tail its logs from the dashboard.
- Anyscale's first-party agent tooling appears to take a non-MCP path.

Across the REP index I didn't see anything about MCP, agents, or an agent-facing
control surface. So the adjacent problems (auth, a human CLI, RayService upgrades)
are clearly being worked — but the "let an **agent** operate Ray" angle seems
absent.

## Why I think it might be worth it (open to being told otherwise)

The thing a generic K8s MCP server *can't* do is the interesting part: surface
Ray's own runtime detail to an agent — live job status, "stuck because no GPU
nodes," logs, follow-a-job-to-completion — which lives in the Ray dashboard/job
API, not in the CRD status. That, plus Ray-aware safety around destructive ops,
feels like a real capability an agent would need and doesn't have today.

## What I'm hoping to learn

1. **Does something like this already exist** (official, ecosystem, or community)
   that I missed? If so, a pointer would be much appreciated.
2. **Is there a reason it shouldn't exist** — a design, security, or
   maintenance concern that's kept it from being built?
3. If it's genuinely an open gap, **would the Ray community want to see it** as a
   community/ecosystem effort, and is there prior discussion I should read first?

Mostly I'd just like to understand the history before going further. Thanks!
