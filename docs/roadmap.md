# Roadmap — AI Gateway Platform

**BLUF.** Ten phases, each independently useful and independently testable. Every capability ships
behind a feature flag defaulting to off, so the minimal stack always deploys clean and the
production stack is the same code with more flags on. No time estimates here — sequencing and exit
criteria only.

**Principle:** a phase is done when its exit criteria are proven by telemetry, not when the
deployment goes green. A green deploy proves the config was accepted. It does not prove the
control works.

---

## Phase 0 — Foundation and agreement `current`

**Goal.** Agree the architecture and what "governed" means before writing infrastructure.

**Ships.** This documentation package — architecture, ten governance areas, executive summary,
this roadmap.

**Exit criteria**
- [ ] Architecture and governance areas signed off
- [ ] Pilot open items answered: network access path, target cloud and region, data classification, audit retention
- [ ] **Data classification answered specifically enough to pick a deployment type** — Global
      Standard, Data Zone, or regional Standard. This one decision moves capacity, cost and region
      strategy, so it cannot wait for phase 5.
- [ ] Phase 1 approved

---

## Phase 1 — Core gateway

**Goal.** One governed model call, end to end, over private networking. Prove the path works
before building governance on top of it.

**Ships**
- `azd` + Bicep, subscription-scope entry point, resource-group-scope modules
- VNet with the full subnet plan, NSGs, NAT, private DNS zones
- APIM, classic Developer tier, internal VNet injection
- One Foundry account, private endpoint only, `publicNetworkAccess=Disabled`, `disableLocalAuth=true`
- Two model deployments: one frontier chat, one mini
- OpenAI-compatible API at `/openai/v1/*` with managed-identity backend auth
- Log Analytics and Application Insights, wired with `metrics: true` and `logAnalyticsDestinationType: Dedicated`
- One access path for the pilot (VPN or peering, per the Phase 0 decision)

**Exit criteria**
- [ ] A call from inside the VNet returns a completion, with the caller holding no Foundry credential
- [ ] Foundry is unreachable except through the gateway — proven from a second VNet resource that
      is *not* the gateway, not assumed from the private endpoint existing
- [ ] `ApiManagementGatewayLogs` shows the request
- [ ] A custom `emit-metric` value lands in Application Insights with a non-zero value
- [ ] `ApiManagementGatewayLlmLog` shows token counts for the same request

**Traps to clear in this phase:** T1, T2, T12, T14, T15, T22 in [architecture.md](architecture.md#13-known-traps-carried-forward).

---

## Phase 2 — Governance controls

**Goal.** Make the gateway actually govern. This is the phase that justifies the platform.

**Ships**
- Product tiers: `sandbox`, `dev-standard`, `dev-power`, `agent-standard`, `agent-production`
- Three stacked throttles per product, keyed on subscription ID
- Logical model aliases resolved to deployments — via APIM's **unified model API** if it is
  available in the target cloud and tier, otherwise hand-rolled in policy
- Per-product model allowlist, and a `/models` endpoint returning the caller's entitled catalog
- Responsible AI policy on every deployment with Jailbreak set to annotate-only
- Content Safety account, private endpoint, gateway policy in **observe mode**
- Azure Consumption budgets at resource-group and subscription scope
- Azure Policy enforcing consistent posture and model versions across Foundry instances
- Remaining-budget response headers so clients can back off

**Exit criteria**
- [ ] A key on `sandbox` is throttled at its configured limit and returns 429 with a remaining-token header
- [ ] A key on `dev-standard` is denied a model outside its allowlist with 403
- [ ] A logical alias resolves to a deployment, and repointing the alias changes the model served with no client change
- [ ] Content safety verdicts appear in telemetry without blocking traffic
- [ ] A budget alert fires on a test threshold
- [ ] Unified model API availability confirmed or ruled out — closes open item O9

**Traps to clear:** T3, T4, T5, T16.

---

## Phase 3 — Client enablement

**Goal.** Real consumers connected, both classes, with documented patterns others can copy.

**Ships**
- VS Code Custom Endpoint configuration for chat-completions and Responses surfaces
- Copilot CLI BYOK wrapper and environment configuration
- Entra app registration exposing the gateway scope, with Azure CLI pre-authorized
- JWT validation mode in policy alongside subscription-key mode
- Reference SDK agent using managed identity, with a documented client pattern
- Request normalization: reasoning-model parameter stripping, `stream_options` handling, Responses api-version handling
- Client onboarding guide

**Exit criteria**
- [ ] A developer runs VS Code Copilot Chat against the gateway and gets a completion
- [ ] A reasoning model works from VS Code without a `400 unsupported_value`
- [ ] Token metrics are captured for a **streamed** response, not just non-streamed
- [ ] An SDK agent authenticates with managed identity and gets a completion, holding no key
- [ ] Telemetry distinguishes developer traffic from agent traffic
- [ ] Decision recorded on whether the stateful Responses surface is in scope — closes open item O11

**Traps to clear:** T6, T7, T8, T9, T10, T13.

---

## Phase 4 — Self-serve onboarding

**Goal.** Remove the human from the provisioning path. Below this line the platform does not scale
past a pilot.

**Ships**
- Onboarding portal — Container App, Entra Easy Auth, private networking
- User-assigned identity with a custom role scoped to APIM subscription CRUD and product read
- Entra group membership driving tier assignment
- Generated client configuration and installer for VS Code and CLI
- Key rotation using primary/secondary keys
- Offboarding and revocation runbook

**Exit criteria**
- [ ] A developer with no prior access signs in, self-registers, and makes a governed model call — no ticket, no admin action
- [ ] Group membership change moves a developer between tiers with no manual APIM work
- [ ] A key is rotated with no service interruption
- [ ] Revoking a subscription immediately blocks that consumer

**Traps to clear:** T11.

---

## Phase 5 — Scale and resilience

**Goal.** Carry production load with an SLA.

**Ships**
- APIM Premium with availability zones (parameter change from Developer)
- **Deployment type decision applied** — Global Standard, Data Zone, or regional Standard, per the
  residency answer from phase 0. This is the real capacity lever, not the pool.
- Second-region Foundry deployment registered as a pool member, same model and same version
- Backend pool with circuit breaker honoring `Retry-After`
- Session-aware distribution for the stateful Responses route
- Dedicated health API reflecting backend availability, not just gateway liveness
- Capacity model and load test results
- Alerting on 429 rate, circuit breaker trips, per-region success rate
- Production access path — corporate network peering

**Exit criteria**
- [ ] Load test at projected concurrent developer count, with sizing evidence for APIM units
- [ ] Killing the primary region's model capacity fails traffic over automatically, proven in telemetry
- [ ] 429 rate stays inside the agreed threshold under load
- [ ] Access path validated on a real corporate laptop image, not a clean test machine
- [ ] Gateway availability target agreed and the single-region trade-off accepted in writing

**Traps to clear:** T17, T18.

**Open items to resolve here:** O1, O2, O8, O10 in [architecture.md](architecture.md#14-open-items).

---

## Phase 6 — Extended model catalog

**Goal.** One gateway for every model type, not just Azure OpenAI.

**Ships**
- Reasoning models (o-series) as a first-class alias
- Anthropic models via the **unified model API** with OpenAI-to-Anthropic format translation —
  not a native `/anthropic` route, which needs a v2 tier we cannot use in Government
- Foundry MaaS open-source models
- Optional self-hosted OSS serving (vLLM or TGI) in-VNet, registered as a gateway backend
- Optional `auto` alias — in-gateway heuristic router between mini and frontier, off by default

**Exit criteria**
- [ ] Every alias in the catalog is callable and reports token usage correctly
- [ ] An OSS backend is governed by the same limits and telemetry as an Azure OpenAI backend
- [ ] Model routing decisions are visible in telemetry with a reason

---

## Phase 7 — Cost attribution and chargeback

**Goal.** Turn token telemetry into something finance and team leads act on.

**Ships**
- Per-model price table and tokens-to-dollars conversion
- Showback workbook pivoting spend by consumer, team, model and month
- Reconciliation against actual Azure spend — gateway token counts are unreliable when streams break
- Cost-center mapping captured at onboarding
- Anomaly alerting on unusual consumer spend
- Optional team-level aggregate caps above individual keys
- **Semantic caching** evaluated against measured traffic, with cache keys scoped by tenant,
  policy context, model version and prompt version

**Exit criteria**
- [ ] A team lead can see their team's monthly AI spend broken down by person and model
- [ ] Reported cost reconciles to the Azure invoice within an agreed tolerance
- [ ] A spend anomaly triggers an alert to a named owner

**Traps to clear:** T19.

**Open item to resolve here:** O6 — showback or true chargeback.

---

## Phase 8 — Evaluation and quality gates

**Goal.** Change models without guessing.

**Ships**
- Regression corpus and evaluation runner as a scheduled job
- Deterministic metrics (exact match, F1, latency, tokens) and AI-assisted metrics (groundedness, relevance, coherence, fluency)
- Results persisted per run for comparison over time
- Promotion gate: an alias only repoints after an eval run passes
- Corpus seeded from real gateway traffic where content logging is permitted — the LLM logs
  export directly as a Foundry evaluation dataset

**Exit criteria**
- [ ] An eval run scores a candidate model against the incumbent and produces a comparable result
- [ ] A model upgrade goes through the gate before the alias moves

---

## Phase 9 — MCP tool-plane governance

**Goal.** Close the gap where a governed model call invokes an ungoverned tool.

**Ships**
- Registry of approved MCP servers with named owners — evaluate **Azure API Center** rather than
  building one
- APIM as an MCP broker — same auth, rate limits and telemetry as model traffic
- In-VNet hosting for internal MCP servers behind private endpoints
- Explicit egress decision and FQDN allow rules for any external MCP server
- Azure Policy blocking agent tools that reach public endpoints, where that is a requirement
- Tool-call audit records

**Exit criteria**
- [ ] An agent's tool calls appear in the same audit stream as its model calls
- [ ] An unapproved MCP server is not reachable from an agent workload

---

## Phase 10 — Second-cloud parity and landing-zone integration

**Goal.** Prove the template is genuinely cloud-portable, and fit it into whatever landing zone
exists by then.

**Ships**
- Deployment to the second cloud from the same template with only a parameter profile change
- Hub VNet peering, corporate DNS integration, Azure Policy compliance
- Mandated naming and tagging standards applied
- CI/CD with OIDC federation for both clouds

**Exit criteria**
- [ ] The same template deploys to both clouds with no Bicep edits
- [ ] The deployment passes the environment's Azure Policy assignments
- [ ] Pipeline deploys to both clouds without stored credentials

---

## Sequencing notes

- **Phases 1–4 are the minimum viable platform.** Anything short of phase 4 cannot absorb 1000+
  developers regardless of how good the gateway is.
- **Phase 5 is the cost decision.** Everything before it runs on the Developer tier. Do not commit
  to Premium until phases 1–4 have proven the value.
- **Phases 6–10 are parallelizable** and can be reordered by demand. Nothing in them blocks the
  others.
- **Phase 2 controls ship in observe mode.** Enforcement is a separate, deliberate switch per tier
  once real usage distribution is known. Dropping hard 429s on the whole fleet on day one is how a
  platform loses its users.
