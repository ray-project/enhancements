# REP: An MCP server for Ray on Kubernetes (KubeRay)

_**Status:** RFC / request for comments. I'd like this to become an **ecosystem
project under `ray-project`** — the REP process asks whether a change belongs
within `ray`, as an ecosystem project under `ray-project`, or as a new project
outside it, and the second is my preference: built KubeRay-native and
upstream-aligned so it could live in the org rather than off to the side. This is
the *case for why* an agent-facing control surface for Ray-on-Kubernetes is worth
having, filed to reach the maintainers who know the history and to invite
correction before I build further. If a lighter venue (Discussion/Slack) is
preferred, happy to move it._

This document is the case. Two companions carry the detail:
- [`2026-06-13-ray-mcp-usecases.md`](2026-06-13-ray-mcp-usecases.md) — concrete
  agent scenarios mapped against each surface. **Start here if you read one file.**
- [`2026-06-13-ray-mcp-design.md`](2026-06-13-ray-mcp-design.md) — full design
  (architecture, tool surface, safety model).

---

## The gap

AI agents are starting to operate infrastructure. For Ray on Kubernetes there is
**no Ray-aware control surface an agent can connect to.** I went looking across OSS
and internally for an MCP (Model Context Protocol) server an agent could use to
create/inspect `RayCluster`s, submit and follow `RayJob`s, and manage
`RayService`s with **Ray-aware semantics**, and couldn't find one that fits.

What exists is adjacent. The `ray-kubectl-plugin` REP is an explicitly
*human-facing* CLI ("for data scientists … unfamiliar with Kubernetes") whose
dashboard story is "open the browser UI." The `kuberay-authentication` REP
addresses dashboard auth — relevant, but not an agent surface. Generic Kubernetes
MCP servers can CRUD the KubeRay CRDs by raw `apiVersion`+`kind` but have zero Ray
awareness. The "let an **agent** operate Ray" angle seems absent. **Is it missing
on purpose, rejected, on a roadmap, or just hasn't come up?**

## Why an agent needs an MCP server, not raw API access

Assume agents operating Ray is a given. The narrower question: why does the agent
need an MCP server rather than being pointed at the Kubernetes API, the KubeRay API
Server, and the Ray dashboard directly?

An agent doesn't browse a dashboard, scroll a terminal, or improvise across APIs —
it calls **tools** and reads results into a **finite context window**. Raw access
gives it context blowout (ingesting raw CRD YAML and 10k-line logs), hand-rolled
cross-plane correlation, no signal for which calls are destructive, and a direct
write path to an unauthenticated, RCE-capable dashboard. An MCP server fixes
exactly these at the protocol layer: typed tools with **bounded, distilled**
results (*"Pending: no GPU nodes,"* not raw YAML); tool annotations
(`readOnlyHint`/`destructiveHint`) the client uses to gate and prompt; the
cross-plane correlation done for it; and **read-only-by-construction** access to
the dashboard.

## Why the KubeRay API Server isn't enough

The KubeRay API Server is useful, but solves a different problem: it is a **CRUD
surface over the CRDs** (the control plane). An agent's hard problems live in the
**data plane** (Ray's dashboard/job API on the head node) and in **cross-plane
correlation** — and a CRUD proxy addresses neither.

- **It doesn't reach Ray's runtime.** Live job status, the *granular* reason a job
  is wedged (unschedulable placement group / no GPU nodes), and job logs live
  behind the **Ray dashboard / Job Submission API (port 8265)** — not in
  `RayJob.status`, which carries only coarse lifecycle + a one-line reason. The API
  Server never touches 8265.
- **By default it doesn't authenticate the caller.** `apiserversdk` is a reverse
  proxy mirroring the Kubernetes API, but as shipped it forwards every request
  under **one shared credential** — it does not pass the agent's identity through
  to Kubernetes RBAC unless the deployer writes custom middleware. "Put the agent
  behind the API Server" does not, by itself, give per-agent authorization.
- **It has no Ray-aware safety.** Nothing distills runtime status for an LLM's
  bounded context, guards Ray-specific footguns (scale-to-zero, deleting a
  RayService that's serving traffic), or predicts CRD field pruning before applying.

A generic K8s MCP server has the same blind spot for the same reason — both stop at
the CRD. So the MCP server an agent needs must be **Ray-aware**.

## Security: you'd write a custom layer regardless

This is the part I most want maintainers to check. The Ray dashboard / Job API is
**unauthenticated by default** — Ray's own guidance treats network isolation as the
primary control, and opt-in token auth only arrived in Ray 2.52.0. It is the
surface behind the **disputed-but-actively-exploited** ShadowRay reports
(CVE-2023-48022; Anyscale's position is that this is intended behavior for a tool
meant to run inside a trusted network). Every scenario that delivers real value to
an agent requires reaching that surface — and a thin CRUD proxy in front of the
CRDs does nothing to make doing so safe.

So whatever you put in front, an agent control surface needs a layer that:

1. **is read-only by construction toward the dashboard** — exposes *no* Ray-side
   write verb, so the unauthenticated/RCE-capable surface is never a mutation
   vector *through the tool* (a property of the tool's interface; it does not make
   8265 itself safe), and
2. **routes every mutation through the guarded, RBAC-gated CRD path** — dry-run +
   diff on every write, tiered registration (`read` always on; `write` /
   `destructive` opt-in), audit log on every mutation.

The destructive-op guards are **agent-safety / anti-footgun**, explicitly *not* a
security control — RBAC is the real boundary. The point stands: even with the
KubeRay API Server in front, you still have to build this Ray-aware + safety layer
on top. That layer is the proposal.

Corrections to anything I've gotten wrong are very welcome — I'd rather be
corrected now than build on a wrong assumption.
