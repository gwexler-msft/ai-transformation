# Governance areas — AI Gateway Platform

**BLUF.** Ten domains define what "governed" actually means for this platform. Each one names the
control, where it is enforced, and what lands in v1 versus later. The rule throughout: if a
control matters, it lives in the gateway. Anything enforced in client code is a suggestion.

Enforcement points referenced below:

| Point | What it is |
|---|---|
| **Gateway** | APIM policy — runs on every request, cannot be bypassed |
| **Model** | Foundry account/deployment configuration — RAI policy, capacity, network posture |
| **Platform** | Azure control plane — RBAC, Policy, budgets, private networking |
| **Client** | Consumer application code — advisory only, never the sole control |

---

## 1. Model catalog, allowlist and approval

**What it covers.** Which models exist, which consumers may call which of them, and how a new
model gets added.

Clients call a **logical alias** (`chat-frontier`, `chat-mini`, `reasoning`, `embeddings`,
`claude`, `oss-*`) that policy resolves to a real deployment. Consumers never name a deployment
directly. This is what makes model upgrades a policy change instead of a 1000-client migration.

| Control | Where | Mechanism |
|---|---|---|
| Only approved models are reachable | Gateway | Alias-to-deployment map. Unknown alias → 400. |
| Per-product model allowlist | Gateway | Product tier determines which aliases resolve. Denied alias → 403. |
| Model discovery reflects entitlement | Gateway | `/models` returns the caller's permitted catalog, not the raw deployment list. |
| New model onboarding | Platform | Bicep parameter + alias mapping + tier assignment. Reviewed change, not a portal click. |
| Version consistency across regions | Platform | Azure Policy enforces the same model and version on every instance an alias resolves to. Drift silently breaks failover. |
| Deprecation | Gateway | Repoint the alias. Optionally return a deprecation header for a grace period before removal. |

**Prefer the platform feature over our own.** API Management's **unified model API** implements
this pattern directly — client-facing aliases, a `/models` discovery endpoint, and format
translation so an Anthropic-shape backend can be called with an OpenAI-shape request. It is in
preview and reaches classic tiers through the AI Gateway early release channel. Verify
availability in the target cloud before deciding to hand-roll.

**Plan for model end-of-life, not just model choice.** Foundation models get retired. The alias
layer is what makes that a gateway change instead of a fleet migration, but it only works if
nothing bypasses it — which is why the private-endpoint NSG in
[architecture.md](architecture.md#8-network-topology) matters as much as the alias map does.

**v1 scope.** Aliases, per-product allowlist, discovery endpoint, catalog defined in IaC.
Approval is a pull request against the catalog parameters.

**Later.** Formal intake workflow with security/legal sign-off recorded against each model;
automatic deprecation notices to affected consumers. **Azure API Center** is worth evaluating
here — it is the Microsoft product for an organizational catalog of APIs, models, MCP servers and
agents with a self-service discovery portal, and it covers a good part of this area plus part of
area 2.

**Open questions.** Who approves a new model — is there an existing AI review board? Does
Government region availability force a different alias map per cloud?

---

## 2. Onboarding, offboarding and key lifecycle

**What it covers.** How a developer or a workload gets a credential, keeps it valid, and loses it.

At 1000+ developers this cannot be a ticket queue. Self-serve is a requirement, not a nice-to-have.

| Control | Where | Mechanism |
|---|---|---|
| Developer self-registration | Platform | Entra-gated portal. Sign in, get a subscription key scoped to your tier. |
| Tier assignment | Platform | Entra security group membership drives which product the key is created against. |
| Least-privilege provisioning | Platform | Portal identity holds a custom role scoped to the APIM service: subscription CRUD and product read. Nothing else. |
| Workload (agent) onboarding | Platform | Reviewed request → one key per workload, owner recorded, tier `agent-standard` or `agent-production`. |
| Key rotation | Gateway | Primary and secondary keys per subscription. Rotate one, leave the other live, cut over, rotate the second. |
| Offboarding | Platform | Leaver process revokes the APIM subscription. Group removal downgrades the tier. |
| Credential hygiene | Client | Keys land in VS Code secret storage or a workload's secret store, never in source. |
| Dormancy | Platform | Keys with no traffic for N days are flagged, then suspended. |

**v1 scope.** Self-serve portal with Entra sign-in and group-driven tiering, primary/secondary key
rotation, manual offboarding runbook.

**Later.** Automated leaver integration with HR/identity lifecycle, dormancy sweep, per-workload
key attestation.

**Open questions.** Does the existing joiner/mover/leaver process expose a hook we can subscribe
to? Is a per-developer key acceptable to security, or does everything need to be JWT?

---

## 3. Throttling, quotas and products

**What it covers.** Bounding what any single consumer can consume, so one runaway agent cannot
starve a thousand developers.

Three throttles stack on every product, all keyed to the subscription:

| Throttle | Protects against | Failure response |
|---|---|---|
| `rate-limit-by-key` (calls/min) | Burst floods, retry storms | 429 |
| `azure-openai-token-limit` (tokens/min) | The real cost driver | 429 with remaining-token header |
| `quota-by-key` (calls/month) | Slow sustained overrun | 403 |

| Control | Where | Mechanism |
|---|---|---|
| Per-consumer limits | Gateway | Product policy, counter keyed on subscription ID |
| Tier differentiation | Gateway | Five products: `sandbox`, `dev-standard`, `dev-power`, `agent-standard`, `agent-production` |
| Client feedback | Gateway | `x-tokens-remaining` / `x-calls-remaining` response headers so clients can back off before being cut off |
| Model-account protection | Model | Deployment `sku.capacity` sized at or above concurrent tier demand |
| Observe-first rollout | Gateway | Limits ship in warn mode, alert only, then flip to enforce per tier |

**v1 scope.** All five products with the proposed limits, enforcement on `sandbox` and
`agent-*` from day one, `dev-*` tiers in warn mode until real usage distribution is known.

**Later.** Team-level aggregate caps above the individual key, time-of-day differentiation,
automatic tier elevation on justified request.

**Open questions.** Are the proposed limits right? They are a starting proposal — one VS Code
chat turn with full workspace context is 30–80k tokens, so anything under ~200k TPM is unusable
for a developer. Validate against a real week of pilot traffic.

---

## 4. Cost attribution and chargeback

**What it covers.** Knowing what each developer, workload and team costs, and deciding whether
they get billed for it.

| Control | Where | Mechanism |
|---|---|---|
| Per-request token capture | Gateway | `llm-emit-token-metric` plus custom `emit-metric` with consumer dimensions |
| Streaming coverage | Gateway | `stream_options.include_usage` injection so streamed calls report usage too — gated on api-version |
| Cost conversion | Platform | Per-model price table, tokens → dollars, in the reporting layer |
| Attribution dimensions | Gateway | `consumer_id`, `consumer_type`, `product`, `model_alias`, `deployment`, `backend_region` |
| Real spend backstop | Platform | `Microsoft.Consumption/budgets` at RG and subscription scope, actual and forecast alerts |
| Reporting | Platform | Workbook pivoting spend by consumer, team, model, month |

**v1 scope.** Token metrics with full dimensions, Azure Consumption budgets, a showback workbook.
Showback only — visibility without internal billing.

**Gateway token counts are not invoice-grade.** API Management captures usage from the model
response, and when a stream breaks or a connection drops that usage can be missing or wrong. Any
chargeback built on token counts needs a reconciliation step against actual Azure spend. Showback
tolerates the gap; real billing does not.

**Later.** Chargeback with real cost transfer, per-team budget enforcement that throttles when a
team's monthly allocation is exhausted, cost anomaly detection, and **semantic caching** — the
largest single cost lever available, deliberately deferred because a badly keyed cache leaks one
user's answer to another. See [architecture.md § 10.1](architecture.md#101-semantic-caching--the-largest-single-cost-lever-we-are-not-using-in-v1).

**Open questions.** Showback or true chargeback? Chargeback needs a cost-center mapping per
consumer, which changes the onboarding data model. Also: is estimated token cost accurate enough
to bill against, or does finance require reconciliation to actual Azure invoice lines?

---

## 5. Observability, audit trail and retention

**What it covers.** Proving what happened. Who called which model, when, with what outcome, and
being able to answer that months later.

| Control | Where | Mechanism |
|---|---|---|
| Request audit record | Gateway | `ApiManagementGatewayLogs` in Log Analytics — one row per call with consumer, product, model, status, latency |
| Token telemetry | Gateway | `ApiManagementGatewayLlmLog` — built-in generative-AI diagnostic capturing prompt, completion and total tokens plus model name, with no policy code |
| Consumer attribution | Gateway | Custom `emit-metric` dimensions layered on top, since the built-in log does not know about our tier and team model |
| Usage dashboard | Platform | Built-in *Language models* analytics dashboard, plus a custom workbook for the dimensions we add |
| Policy decision visibility | Gateway | Emitted dimensions for route decision, throttle reason, safety verdict |
| Backend health | Platform | Circuit breaker state, per-region success rate, capacity 429 rate |
| Alerting | Platform | 429 rate, token spend thresholds, backend failure, safety block rate |
| Retention | Platform | Log Analytics retention policy plus archive tier for the audit window |

**Prompt and completion content is not logged by default.** That is a deliberate default, not an
oversight — see [Data residency, DLP and PII](#7-data-residency-dlp-and-pii). Metadata is always
logged. Content capture **is available** as a separate opt-in on the same diagnostic (32 KB per
entry, chunked above that, 2 MB ceiling), so if audit requirements later demand full interaction
records, the mechanism exists and does not need building. Those same logs export directly as an
evaluation dataset for Foundry, which links this area to area 9.

Two configuration flags decide whether any telemetry exists at all, and both fail silently:

- APIM `applicationinsights` diagnostic needs `metrics: true` or every `emit-metric` is skipped.
- The Log Analytics diagnostic setting needs `logAnalyticsDestinationType: 'Dedicated'` or the
  gateway log table stays empty.

**v1 scope.** Gateway logs, token metrics, an operations workbook, alerts on 429 rate and backend
health. Default retention.

**Later.** Immutable audit export, long-term archive to storage with legal hold, SIEM integration,
SLO dashboards.

**Open questions.** What is the required audit retention period? Does it need to be immutable or
exportable to an existing SIEM? Is there a compliance framework driving this (FedRAMP, CMMC,
internal)?

---

## 6. Content safety, Prompt Shields and responsible AI

**What it covers.** Screening prompts and responses for harmful content and prompt-injection
attempts, and configuring the model's own responsible-AI filters correctly.

Two layers:

| Layer | Where | Mechanism |
|---|---|---|
| Gateway screening | Gateway | `llm-content-safety` policy against a private Content Safety account — Hate, Sexual, SelfHarm, Violence plus Prompt Shields. Harmful prompt → 403 before the model is reached. |
| Model-side RAI | Model | Responsible AI policy attached to every deployment |

**The RAI policy must not be `Microsoft.DefaultV2`.** Its Jailbreak shield is set to blocking, and
it reliably rejects VS Code Copilot's own system prompts — tool-calling preambles and instruction
framing look like jailbreak attempts. The result is `400 content_filter` on completely benign
developer requests. Use a policy with Jailbreak set to annotate-only. Annotate-only does not
require a modified-content-filter approval; disabling a category does.

A second trap: if the Content Safety account has `disableLocalAuth=true`, the APIM backend must
carry a managed-identity credential. RBAC alone is not enough, and without it **every** prompt
fails closed with 403 — including benign ones. That symptom looks like a broken safety config,
not a broken auth config, and it costs hours.

| Control | Where | Mechanism |
|---|---|---|
| Prompt screening | Gateway | Content Safety with configurable severity threshold per category |
| Jailbreak detection | Gateway + Model | Prompt Shields at the gateway, annotate-only at the model |
| Response screening | Gateway | Outbound content safety on non-streamed responses |
| Safety telemetry | Gateway | Block rate and category emitted as metrics for tuning |
| Per-tier strictness | Gateway | Thresholds parameterized per product — a sandbox tier can be stricter than a production agent |

**v1 scope.** Model-side RAI policy with annotate-only Jailbreak on all deployments. Gateway
content safety deployed but in **observe mode** — emitting verdicts without blocking — until false
positive rates on real developer traffic are known.

**Later.** Enforcement on inbound, response screening, per-tier thresholds, custom blocklists.

**Open questions.** What false-positive rate is acceptable for developer traffic? Coding prompts
trip safety filters more than general chat. Does streaming response screening matter, given it
adds latency and cannot fully screen a stream?

---

## 7. Data residency, DLP and PII

**What it covers.** What data is allowed to reach a model, where it physically goes, and what is
retained.

| Control | Where | Mechanism |
|---|---|---|
| Residency | Platform | Foundry deployed only in approved regions. Cloud and region are deployment parameters. |
| **Processing residency** | Model | **Deployment type.** Global Standard keeps data at rest in the geography but may process inference in any hosting location. Data Zone constrains processing to a defined zone. Regional Standard pins both. |
| Network containment | Platform | Model accounts have no public endpoint. Traffic never leaves the VNet or the private backbone. |
| No content logging by default | Gateway | Metadata only. Prompt and completion bodies are not written to logs. |
| Zero data retention | Model | Foundry deployments configured so prompts are not retained for abuse monitoring where the offer supports it |
| PII detection | Gateway | Optional — PII detection on the prompt before forwarding, redact or block |
| Egress control | Platform | Gateway subnet default-deny with an explicit allow list, NAT for a deterministic outbound IP |

**Deployment type is where residency and capacity collide.** Global Standard is Microsoft's
answer to throttling at scale, and it is also the option that permits inference processing outside
the deployment region. If the data classification forbids that, we lose the easy capacity answer
and move to regional deployments with overcommit and probably provisioned throughput. This is one
question with a large blast radius across cost, capacity and region strategy — open item **O8**.

**No cross-boundary routing.** Any gateway that load-balances across a geopolitical boundary
breaks residency, however good the failover story is. If residency applies, each boundary gets its
own independent gateway and clients are blocked from reaching any other one.

**The unsolved edge: GitHub Copilot's own telemetry.** BYOK re-points *inference* to the private
gateway. The IDE still talks to GitHub for auth, entitlement and telemetry. VS Code has no master
switch to stop this. If that egress is unacceptable, it has to be blocked at the network layer
with an FQDN-aware control — Azure Firewall application rules or a secure web gateway. NSGs
cannot do it because they operate at layer 4.

**v1 scope.** Private-only model access, approved-region deployment, no content logging, egress
allow-listing on the gateway subnet.

**Later.** PII detection and redaction in policy, DLP pattern matching, documented Copilot
telemetry egress posture with an FQDN blocklist if required.

**Open questions.** What is the data classification of what developers will actually put in
prompts — source code, customer data, anything regulated? Is GitHub Copilot's telemetry egress
acceptable, and if not, does an FQDN-capable egress control already exist in the environment?

---

## 8. Resilience, multi-region and failover

**What it covers.** Staying available when a region, a deployment, or model capacity fails.
Capacity 429s are the normal failure mode at this scale, not an exception.

| Control | Where | Mechanism |
|---|---|---|
| Multi-region model backends | Gateway | Foundry deployments in two or more regions registered as pool members |
| Automatic failover | Gateway | Backend pool with `priority` (active/passive), `weighted` (active/active), round-robin or session-aware distribution |
| Fast capacity spill | Gateway | Circuit breaker per pool member, honoring `Retry-After` so 429s move traffic promptly instead of retrying into a wall |
| Version consistency | Platform | Every pool member serves the same model **and version**. Enforced by Azure Policy. |
| Session affinity | Gateway | Session-aware distribution for the stateful Responses API, so a follow-up lands on the backend holding the conversation |
| Gateway availability | Platform | APIM Premium with availability zones and optional multi-region deployment |
| Graceful degradation | Gateway | Optional fallback from a saturated frontier model to a mini model rather than failing the request |
| Backend health visibility | Gateway | Dedicated health API — APIM's default status endpoint reports the gateway only, not the models behind it |
| Capacity headroom | Model | Deployment type first, then capacity sized above concurrent tier caps, then PTU where latency guarantees matter |

**Three things that are easy to get wrong here:**

1. **A pool does not add quota.** Standard quota is subscription-scoped. More instances in the
   same subscription means more availability and exactly the same throughput ceiling. Capacity
   comes from the deployment type or from spreading across subscriptions.
2. **A single-region gateway in front of multi-region backends is a global single point of
   failure.** It survives a model outage, not a gateway outage. That is an acceptable trade for a
   developer productivity platform. It is not acceptable for anything business-critical, and if
   the SLO changes the answer is multi-region APIM — noting the control plane stays single-region
   even then.
3. **Never fail over between model versions.** Routing a retry from version *X* to *X+1* produces
   behaviour changes users cannot explain and we cannot reproduce.

**v1 scope.** Multi-region pool with circuit breaker from day one, as decided. Single APIM
instance during pilot; Premium with zones for production.

**Later.** Multi-region APIM, active health probes against pool members, automated capacity
scaling, PTU for production agents.

**Open questions.** What is the availability target — is this on the critical path for anything,
or is degraded developer productivity an acceptable outage? Which Government region pairs have the
required models available in both?

---

## 9. Evaluation and quality gates

**What it covers.** Knowing whether a model change made things better or worse before 1000
developers find out.

| Control | Where | Mechanism |
|---|---|---|
| Regression corpus | Platform | A fixed test set run against every candidate model or policy change |
| Deterministic metrics | Platform | Exact match, token F1, latency, token count |
| AI-assisted metrics | Platform | Groundedness, relevance, coherence, fluency scored by a judge model through the gateway |
| Scheduled runs | Platform | Container Apps Job, manual trigger or cron |
| Results store | Platform | Results persisted per run so changes are comparable over time |
| Promotion gate | Platform | An alias only repoints to a new deployment after the eval run passes |

This is what makes decision D5 (logical aliases) safe. Repointing `chat-frontier` to a new model
is a low-risk operation *only if* something proves the new model is not worse first.

**v1 scope.** Not in v1. The eval harness exists as a proven pattern from prior work and can be
lifted in once the catalog has more than one candidate per alias.

**Later.** Full eval runner, promotion gate wired into the alias change process, per-team custom
corpora.

**Open questions.** Who owns the regression corpus, and does it need domain-specific test cases
per team? Does an eval failure block a model upgrade, or just warn?

---

## 10. MCP tool-plane governance

**What it covers.** Model Context Protocol servers — the tools agents call. This is a **separate
plane** from model inference and it is not governed by the model gateway by default.

This matters more than it looks. A governed model call that invokes an ungoverned tool that
reaches an arbitrary internet endpoint has moved the risk, not removed it.

| Control | Where | Mechanism |
|---|---|---|
| Tool inventory | Platform | Registry of approved MCP servers, who owns each, what it reaches. **Azure API Center** is the Microsoft product for this. |
| Brokered access | Gateway | APIM fronts MCP servers with the same subscription-key auth, rate limits and telemetry as model traffic |
| Private hosting | Platform | Internal MCP servers hosted in-VNet behind private endpoints |
| Remote MCP egress | Platform | Any external MCP server is an explicit egress decision with an FQDN allow rule |
| Tool-call telemetry | Gateway | Which consumer called which tool, when, with what result |

**The tool plane is moving fast.** API Management can already expose REST APIs as MCP servers and
govern existing ones, and Foundry's control plane can register agents and MCP tools for central
inventory and policy. If we adopt Foundry Agent Service later, note that several of its tools —
Bing Grounding, web search, SharePoint grounding — reach **public endpoints** even when the
Foundry resource is fully network-isolated. That is a policy decision, not a networking one, and
it can be blocked with Azure Policy.

**v1 scope.** Documented position only: the model plane is governed, the tool plane is not yet,
and any MCP usage during the pilot is an explicit exception with a named owner.

**Later.** APIM as an MCP broker, approved-server registry, in-VNet hosting for internal tools,
tool-call audit.

**Open questions.** Are teams already using MCP servers, and which ones? Is remote MCP (for
example a hosted GitHub MCP server) acceptable, or must everything be in-tenant?

---

## Summary — what lands in v1

| Area | v1 | Later |
|---|---|---|
| 1. Model catalog and allowlist | Aliases, per-product allowlist, discovery endpoint | Formal intake workflow |
| 2. Onboarding and key lifecycle | Self-serve portal, group-driven tiers, key rotation | Automated leaver, dormancy sweep |
| 3. Throttling and quotas | Five products, enforce on agents, warn on developers | Team-level caps, auto tier elevation |
| 4. Cost attribution | Token metrics, budgets, showback workbook | True chargeback, per-team enforcement |
| 5. Observability and audit | Gateway logs, token metrics, alerts, workbook | Immutable archive, SIEM integration |
| 6. Content safety and RAI | Annotate-only Jailbreak RAI on all deployments, safety in observe mode | Enforcement, response screening, per-tier thresholds |
| 7. Residency, DLP, PII | Private-only, approved regions, no content logging | PII redaction, DLP patterns, Copilot egress posture |
| 8. Resilience | Multi-region pool, circuit breaker | Multi-region APIM, health probes, PTU |
| 9. Evaluation | Not in v1 | Eval runner, promotion gate |
| 10. MCP tool plane | Documented position, exceptions by name | APIM as MCP broker, approved registry |
