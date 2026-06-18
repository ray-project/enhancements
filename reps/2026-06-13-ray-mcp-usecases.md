# ray-mcp — Use Cases: why an agent needs an MCP server for Ray

**Read time: ~10 minutes.** This assumes agents operating Ray is a given. The
question it answers is narrower: **why does the agent need an MCP server** — a
Ray-aware one — rather than being pointed at the Kubernetes API, the KubeRay API
Server, and the Ray dashboard directly?

Companion to [`2026-06-13-ray-mcp-rfc.md`](2026-06-13-ray-mcp-rfc.md) (the case)
and [`2026-06-13-ray-mcp-design.md`](2026-06-13-ray-mcp-design.md) (the design).

## Why an MCP server, not raw API access

An agent doesn't browse a dashboard, scroll a terminal, or improvise across APIs.
It calls **tools** and reads results into a **finite context window**. Hand it raw
access to the three Ray surfaces and you get:

- **Context blowout + brittle parsing** — it ingests raw CRD YAML and 10k-line logs
  and reasons over them itself.
- **Hand-rolled cross-plane correlation** — "live status" means stitching CRD →
  submission id → dashboard, every time, correctly.
- **A loaded gun** — direct write access to an unauthenticated, RCE-capable
  dashboard.
- **No danger signal** — the client has no way to know which calls are destructive.

An MCP server fixes exactly these, at the protocol layer:

- **Typed tools + bounded, distilled results** — the agent gets *"Pending: no GPU
  nodes,"* not raw YAML; a byte-capped log tail, not a firehose.
- **Tool annotations** (`readOnlyHint` / `destructiveHint`) the MCP client uses to
  gate and prompt — free protocol-layer safety.
- **The correlation is done for it** — one tool call spans the planes.
- **Read-only by construction toward the dashboard** — the tool exposes no Ray-side
  write verb; mutations go only through the guarded CRD path.

That's why "agent + MCP server" beats "agent + raw APIs." The rest of this doc
shows why that MCP server must be **Ray-aware** — a generic Kubernetes MCP gives
typed CRD tools but is still blind to Ray's runtime.

## How the existing surfaces compare

| | **Generic K8s MCP** | **KubeRay API Server** | **`kubectl ray` plugin** | **Ray-aware MCP (proposed)** |
|---|---|---|---|---|
| Consumer | agent | agent | **human at a terminal** | agent |
| Layer | k8s API (any CRD) | CRUD over KubeRay CRDs | human CLI over CRDs + port-forward | CRD path **+** Ray dashboard |
| Reaches Ray dashboard (:8265) | ❌ | ❌ | ✅ port-forward² | ✅ read-only |
| "Why is this job stuck?" | ❌ raw CRD status | ❌ raw CRD status | ❌ status tables | ✅ distilled |
| Live job logs | ❌ | ❌ | partial (download to disk³) | ✅ (bounded tail) |
| Submit → follow to completion | ❌ | partial (CRD only) | ✅ (`job submit`) | ✅ cross-plane |
| Ray-aware typed params / pruning warnings | ❌ | partial | partial (typed flags, no pruning warn) | ✅ |
| Ray-specific destructive guards | ❌ | ❌ | ❌ | ✅ (agent-safety) |
| Machine-consumable contract (typed tools, danger hints) | ✅ (generic) | partial (REST) | ❌ human tables | ✅ Ray-aware |
| Caller authn/authz by default | inherits kubeconfig | ❌ shared credential¹ | inherits kubeconfig | inherits kubeconfig/SA RBAC |

¹ The KubeRay API Server (`apiserversdk`) proxies the Kubernetes API but, as
shipped, forwards requests under **one shared credential** — it doesn't pass the
agent's identity through to Kubernetes RBAC unless the deployer adds custom
middleware.

² `kubectl ray session` *port-forwards* the dashboard to `localhost:8265`; it
**opens** the surface, it doesn't consume it. An agent on top would still drive raw
HTTP against the unauthenticated, RCE-capable dashboard itself — the loaded gun this
doc is about. The proposed server reads that surface *for* the agent, read-only by
construction.

³ `kubectl ray log` downloads head/worker logs to a local directory, and `job
submit` prints job output inline until completion — neither is a byte-bounded tail
an agent can pull into a finite context window on demand.

**The pattern:** the generic K8s MCP and the KubeRay API Server both stop at the CRD,
so the agent's hard problems — data-plane truth and cross-plane correlation — are out
of reach. `kubectl ray` *does* reach the runtime, but as a **human onboarding CLI**
([REP #52](https://github.com/ray-project/enhancements/pull/52)): it port-forwards
the dashboard rather than distilling it, emits tables rather than a typed contract,
and has no agent-safety guards. An agent built on top of it would still have to add
the distillation + safety layer — which is this proposal. The two are complementary:
`kubectl ray` is the paved **human** path; this is the **agent** path.

---

## Scenario 1 — "My training job is stuck. Why?"

`RayJob.status` gives the agent `Pending` and maybe a one-line reason — not *why*.
The real cause (autoscaler can't place a GPU bundle, unschedulable placement group,
image pull backoff) lives in pod events and the **Ray dashboard's resource view**,
not the CRD. A generic K8s MCP / KubeRay API Server leaves the agent guessing or
dumping raw YAML into its context. **Ray-aware MCP:** `ray_job_get` returns a
distilled, actionable status — *"Pending: unschedulable — 0/1 GPU nodes"* — a
sentence, not a blob. This "distill the runtime truth" loop is the single most
common agent task, and a CRD-only surface structurally cannot do it.

## Scenario 2 — "Submit this job and tell me when it actually finishes."

One logical operation spanning **two control planes**: create the RayJob CRD, then
track the run to terminal with logs. A CRUD surface creates the CRD and then goes
blind — terminal status and logs are in the dashboard, keyed by a **submission id**
the agent doesn't have. **Ray-aware MCP:** bridges RayJob name → `status.jobId` →
dashboard automatically; submit returns immediately (no multi-hour blocking call
that would hit the client timeout), and the agent follows with a *bounded*
`ray_job_wait` + `ray_job_logs`.

## Scenario 3 — "Tail the last 200 lines of job `nightly-train`."

Logs aren't in the CRD — they're at `GET /api/jobs/{submission_id}/logs` on the
dashboard. Generic K8s MCP / KubeRay API Server: ❌. **Ray-aware MCP:** resolves
name → submission id, reaches the dashboard, returns a **byte-bounded** tail. Logs
are table-stakes for debugging and structurally unreachable from the CRD.

## Scenario 4 — "Scale the worker group" / "deploy a RayService update."

Deep, nested KubeRay specs are exactly where agents fail. A generic tool hand-builds
nested paths, gets them subtly wrong, and the API server may **silently prune**
unknown fields (no error — the agent thinks it worked); it also races the
**autoscaler**, which writes `replicas` directly. **Ray-aware MCP:** typed
worker-group params; **Server-Side Apply** that respects autoscaler field
ownership; **pruning prediction** from the installed CRD schema (warns *before*
applying); and it knows that editing `serveConfigV2` is an in-place Serve update
while editing cluster config triggers a zero-downtime cluster swap.

## Scenario 5 — "Clean up the old clusters."

To a CRUD surface, a delete is a delete — no notion that a RayService is *currently
serving traffic*, or that deleting an *ephemeral* RayJob cascade-deletes a whole
cluster. **Ray-aware MCP:** refuses to delete a serving RayService unless forced;
treats scale-to-zero and cascade-deletes as destructive; requires a content-derived
**confirm-fingerprint** that also catches the resource changing between preview and
commit. **Honest caveat:** these guards are **agent-safety / anti-footgun**, *not* a
security control — **RBAC is the security boundary.** They exist because an
autonomous agent has no human instinct for "that one's dangerous."

---

## The security throughline

Every scenario that delivers real value (1, 2, 3) requires reaching the **Ray
dashboard / Job API on :8265** — which is **unauthenticated by default** (Ray's
guidance: network isolation is the primary control; opt-in token auth arrived only
in Ray 2.52.0) and is the surface behind the **disputed-but-actively-exploited**
ShadowRay reports (CVE-2023-48022; Anyscale considers it intended behavior for a
tool meant to run in a trusted network).

This is the crux of why an agent needs an MCP server here and not raw access: the
MCP layer is what lets the agent get the dashboard's value **without** holding a
write path to it. The server:

1. **is read-only by construction toward the dashboard** — exposes no Ray-side write
   verb at all, so it can never be a mutation/RCE vector *through the tool* (a
   property of the tool's interface; it does not make :8265 itself safe), and
2. **routes every mutation through the guarded, RBAC-gated CRD path** — dry-run +
   diff on every write, tiered registration (`read` always on; `write` /
   `destructive` opt-in), audit log on every mutation.

Point the agent at the raw APIs and you've handed it that loaded gun directly. The
MCP server is what makes the dashboard's runtime detail usable *and* safe — and it
has to be Ray-aware to do either.
