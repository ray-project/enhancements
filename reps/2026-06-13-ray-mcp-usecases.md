# ray-mcp — Use Cases: what an agent needs, and why existing surfaces fall short

**Read time: ~10 minutes.** This is the evidence behind the RFC. It walks concrete
things an AI agent is asked to do with Ray on Kubernetes, and shows — per scenario
— what a **generic Kubernetes MCP server** and the **KubeRay API Server** can and
cannot do, and what a Ray-aware MCP layer adds.

Companion to [`2026-06-13-ray-mcp-rfc.md`](2026-06-13-ray-mcp-rfc.md) (the case)
and [`2026-06-13-ray-mcp-design.md`](2026-06-13-ray-mcp-design.md) (the design).

## The three surfaces compared

| | **Generic K8s MCP** | **KubeRay API Server** | **Ray-aware MCP (proposed)** |
|---|---|---|---|
| Layer | k8s API (any CRD) | CRUD over KubeRay CRDs | CRD path **+** Ray dashboard |
| Reaches Ray dashboard (:8265) | ❌ | ❌ | ✅ read-only |
| "Why is this job stuck?" | ❌ raw CRD status | ❌ raw CRD status | ✅ distilled |
| Live job logs | ❌ | ❌ | ✅ (bounded tail) |
| Submit → follow to completion | ❌ | partial (CRD only) | ✅ cross-plane |
| Ray-aware typed params / pruning warnings | ❌ | partial | ✅ |
| Ray-specific destructive guards | ❌ | ❌ | ✅ (agent-safety) |
| Caller authn/authz by default | inherits kubeconfig | ❌ shared credential¹ | inherits kubeconfig/SA RBAC |

¹ V1 (deprecated) is unauthenticated over NodePort + optional shared static token.
V2 `apiserversdk` (alpha) proxies the k8s API but forwards requests under **one
shared credential** unless the deployer adds custom middleware — it does not pass
the agent's identity through to Kubernetes RBAC by default.

**The pattern:** the things a generic K8s MCP server *can't* do are the same things
the KubeRay API Server *can't* do — because both stop at the CRD. The agent's hard
problems live in Ray's data plane and in cross-plane correlation.

---

## Why an agent specifically needs this (vs. a human + CLI)

A human debugging a stuck job opens the dashboard, eyeballs the resource panel,
tails logs in a terminal, and *knows* not to delete the cluster that's serving
prod. An agent has none of that: it has tools and a finite context window.

- It can't "open the browser UI" — it needs the dashboard's data as a **tool
  result**.
- It can't scroll a 10,000-line log — it needs a **bounded, distilled** answer or
  it blows its context budget and degrades.
- It doesn't have a human's instinct for "this delete is dangerous" — danger has
  to be **encoded** (tiered guards, confirm-on-destructive, read-only-by-construction
  toward the RCE-capable dashboard).

That's why the existing human-facing path (`ray-kubectl-plugin`, "open the
dashboard") doesn't transfer, and why a generic CRUD surface isn't enough.

---

## Scenario 1 — "My training job is stuck. Why?"

**Agent's job:** submit a `RayJob`, notice it's not progressing, explain why, fix it.

- **Generic K8s MCP / KubeRay API Server:** can read `RayJob.status` →
  `jobDeploymentStatus: Pending`, maybe a one-line `reason`. It cannot say *why*.
  The real cause — autoscaler can't place a GPU bundle, unschedulable placement
  group, image pull backoff on the worker — lives in pod events and the **Ray
  dashboard's resource view**, not the CRD. The agent is left guessing or dumping
  raw YAML into its context.
- **Ray-aware MCP:** `ray_job_get` returns a **distilled, agent-actionable** status:
  *"Pending: unschedulable — 0/1 GPU nodes available"* plus the relevant pod event.
  The agent acts on a sentence, not a YAML blob.

**Why it matters:** "distill the runtime truth" is the single most common agent
loop, and it's exactly what a CRD-only surface cannot do.

---

## Scenario 2 — "Submit this job and tell me when it actually finishes."

**Agent's job:** one logical operation that spans **two control planes** — create the
RayJob CRD, *then* track the run to a terminal state with logs.

- **Generic K8s MCP:** creates the CRD. Then it's blind — terminal job status and
  logs are in the dashboard, keyed by a **submission id** the agent doesn't have.
- **KubeRay API Server:** creates the CRD and exposes CRD-level status, but still
  doesn't reach the dashboard for logs or live run detail.
- **Ray-aware MCP:** bridges **RayJob name → `status.jobId` (submission id) →
  dashboard endpoint** automatically. Submit returns immediately (no multi-hour
  blocking call that would hit the MCP client timeout); the agent follows with a
  *bounded* `ray_job_wait` and `ray_job_logs`. The cross-plane correlation is the
  whole point.

**Why it matters:** this is the cross-plane workflow neither CRUD surface performs;
getting the submit/follow lifecycle right (non-blocking, bounded) is what makes it
usable by a real agent on a real client timeout.

---

## Scenario 3 — "Tail the last 200 lines of job `nightly-train`."

- **Generic K8s MCP / KubeRay API Server:** ❌ — logs are not in the CRD. They live
  at `GET /api/jobs/{submission_id}/logs` on the dashboard.
- **Ray-aware MCP:** resolves name → submission id, reaches the dashboard
  (in-cluster DNS or an on-demand port-forward), returns a **byte-bounded** tail.

**Why it matters:** logs are table-stakes for debugging, and structurally
unreachable from the CRD.

---

## Scenario 4 — "Scale the worker group" / "deploy a RayService update."

**Agent's job:** edit deep, nested KubeRay specs correctly — where agents reliably fail.

- **Generic K8s MCP:** raw spec surgery. The agent hand-builds nested paths, gets
  them subtly wrong, and the API server may **silently prune** unknown fields — no
  error, the agent believes it worked. It also races the **autoscaler**, which
  writes `replicas` directly (a naive get-modify-put clobbers it).
- **KubeRay API Server:** typed-ish create, but still no pruning warning, no
  autoscaler-safe apply, no awareness that editing `serveConfigV2` is an *in-place*
  Serve update while editing the cluster config triggers a *zero-downtime cluster
  swap*.
- **Ray-aware MCP:** typed worker-group params; **Server-Side Apply** that respects
  the autoscaler's field ownership; **pruning prediction** from the installed CRD
  schema (warns *before* applying); reports which RayService update path a change
  takes.

**Why it matters:** these are Ray-specific correctness traps. A generic tool gets
them wrong silently; an agent can't tell.

---

## Scenario 5 — "Clean up the old clusters."

**Agent's job:** delete safely — without taking down something that matters.

- **Generic K8s MCP / KubeRay API Server:** a delete is a delete. No notion that a
  RayService is *currently serving traffic*, or that deleting an *ephemeral* RayJob
  cascade-deletes a whole cluster.
- **Ray-aware MCP:** Ray-tuned destructive tier — refuses to delete a serving
  RayService unless forced; treats scale-to-zero and cascade-deletes as
  destructive; requires a content-derived **confirm-fingerprint** that also catches
  the resource changing between preview and commit.

**Why it matters (and the honest caveat):** these guards are **agent-safety /
anti-footgun**, *not* a security control — **RBAC is the security boundary.** They
exist because an autonomous agent has no human instinct for "that one's
dangerous." That's a real need, but we don't oversell it as security.

---

## The security throughline

Every scenario that delivers real value (1, 2, 3) requires reaching the **Ray
dashboard / Job API on :8265** — which is **unauthenticated by default** (Ray's
guidance: network isolation is the primary control; opt-in token auth arrived only
in Ray 2.52.0) and is the surface behind the **disputed-but-actively-exploited**
ShadowRay reports (CVE-2023-48022; Anyscale considers it intended behavior for a
tool meant to run in a trusted network).

A thin CRUD proxy in front of the CRDs does nothing to make reaching that surface
safe. An agent-facing layer has to:

1. **be read-only by construction toward the dashboard** — expose no Ray-side write
   verb at all, so the tool can never be a mutation/RCE vector *through it* (a
   property of the tool's interface; it does not make :8265 itself safe), and
2. **route every mutation through the guarded, RBAC-gated CRD path** — dry-run +
   diff on every write, tiered registration (`read` always on; `write` and
   `destructive` opt-in), audit log on every mutation.

This is why "just put the KubeRay API Server in front of the agent" doesn't close
the gap: **you still have to build the Ray-aware + safety layer on top.** That layer
is the proposal — and the place I'd most value the maintainers' review.
