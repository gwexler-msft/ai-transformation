# Executive summary — AI Gateway Platform

**BLUF.** Build one governed front door for every AI model call in the enterprise. Azure API
Management acts as an AI gateway in front of Azure AI Foundry, so identity, spend, safety and
audit are enforced once, centrally, instead of being reinvented — or skipped — by every team.

## The problem

AI adoption is happening whether or not it is governed. Teams are standing up their own model
access, developers want GitHub Copilot pointed at company-controlled models, and agents are being
built against direct model endpoints. Today that means:

- **No spend control.** Nobody can answer what AI costs, per team or per person, until the invoice.
- **No consistent guardrails.** Content safety and responsible-AI configuration is per-project.
- **No audit trail.** No single record of who called which model with what outcome.
- **Credential sprawl.** Model keys distributed to applications and developers.
- **No portability.** Every project hard-codes a model version, so upgrades are fleet migrations.

## The approach

Every model call goes through one gateway. Consumers hold a credential scoped to a tier and
nothing else — never a model key, never a model endpoint.

| We enforce | Where |
|---|---|
| Who can call | Subscription key or Entra identity, validated at the gateway |
| Which models they can call | Per-tier allowlist of logical model aliases |
| How much they can consume | Rate limit, token limit and monthly quota per consumer |
| What content is allowed | Content safety and Prompt Shields before the model is reached |
| What it cost and who owes it | Per-consumer token telemetry converted to dollars |
| What happened | One audit record per request |

Model accounts have no public endpoint and no key-based access. The gateway's managed identity is
the only thing that can open them.

## What it serves

Two consumer classes, one gateway, one policy, one telemetry stream:

- **Developers** — VS Code with GitHub Copilot BYOK, and Copilot CLI, pointed at company models.
- **Agents and services** — SDK workloads calling models programmatically with managed identity.

Designed for **1000+ developers and many teams**, deployable to **Azure Government or Commercial**
from the same template with a parameter change.

## Delivery shape

Documentation first — this package — then infrastructure as code in phases. Each phase is
independently useful and independently fundable.

| | Phase | Outcome |
|---|---|---|
| 1 | Core gateway | First governed model call end to end. Proves the path. |
| 2 | Governance controls | Safety, model allowlist, tiers and limits enforced. |
| 3 | Client enablement | Developers and agents connected through documented patterns. |
| 4 | Self-serve onboarding | Scales past what a ticket queue can absorb. |
| 5 | Scale and resilience | Production SLA, multi-region, capacity planning. |
| 6+ | Extended catalog, chargeback, evaluation, tool governance | Depth once the platform is load-bearing. |

## Cost shape

Two components dominate. Everything else is noise by comparison.

- **The gateway.** Pilot runs on the Developer tier for tens of dollars a month. Production
  requires the Premium tier — thousands of dollars per unit per month — because it is the only
  classic tier with a private-network SLA, and the v2 tiers are not available in Government.
  This step is the single largest decision point in the programme.
- **Model consumption.** Scales with adoption. This is exactly what the platform exists to make
  visible and bounded, and the reason the token limits and budgets ship in phase 2 rather than later.

Everything else — networking, telemetry, the onboarding service — is a rounding error against
those two.

## Risks worth naming now

| Risk | Mitigation |
|---|---|
| Private gateway is unreachable from developer laptops | Three access paths designed in: corporate network, VPN, in-VNet desktops. Decide early — it gates the pilot. |
| Limits set too tight, developers abandon the platform | Every limit ships in observe-only mode first. Enforce after real usage is measured. |
| Data classification forces region-locked model processing | Azure's higher-capacity model deployment options allow inference to be processed outside the deployment region. If our data cannot permit that, capacity gets harder and more expensive. **This is the single decision with the widest blast radius** and it is needed before the pilot, not after. |
| Model capacity cannot meet 1000+ developers | Multi-region failover from day one, plus reserved capacity for production workloads if required. Capacity model is an open item. |
| Gateway cost at production scale | Phased. Pilot proves value before the Premium tier is committed. |

## The ask

1. **Agree the architecture and the ten governance areas** in the accompanying documents.
2. **Answer the open items** that gate the pilot — network access path, target cloud and region,
   data classification, audit retention requirement.
3. **Approve phase 1** so infrastructure work can start.

## Documents in this package

- [architecture.md](architecture.md) — target architecture, diagrams, components, known traps
- [governance-areas.md](governance-areas.md) — the ten governance domains in detail
- [microsoft-guidance-alignment.md](microsoft-guidance-alignment.md) — the design reviewed against Microsoft's published guidance
- [roadmap.md](roadmap.md) — phased delivery plan with exit criteria

The design has been checked against Microsoft's current reference architecture and Well-Architected
guidance for AI gateways. The core shape held; six things changed as a result, and they are
recorded rather than quietly absorbed.
