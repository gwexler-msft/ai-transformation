# AI Gateway Platform

A centralized AI governance platform for Azure. One governed front door — **Azure API Management
as an AI gateway** — in front of **Azure AI Foundry** models, so every model call from every
consumer is authenticated, rate-limited, safety-screened, attributed to a cost owner, and logged.

Consumers are **SDK agents** (services calling models programmatically) and **developer IDE
surfaces** (VS Code + GitHub Copilot BYOK, Copilot CLI). Both hit the same gateway, under the same
policy, with the same telemetry.

## Status

**Documentation phase.** No infrastructure yet. The docs below define the target design and are
the artifact under review. Bicep follows agreement.

## Docs

| Doc | What's in it | Audience |
|---|---|---|
| [docs/executive-summary.md](docs/executive-summary.md) | One page: problem, approach, phasing, cost shape, the ask | Leadership |
| [docs/architecture.md](docs/architecture.md) | Target architecture, diagrams, components, network, IaC approach, known traps | Engineering |
| [docs/governance-areas.md](docs/governance-areas.md) | The ten governance domains — what we enforce and where | Engineering + security + finance || [docs/microsoft-guidance-alignment.md](docs/microsoft-guidance-alignment.md) | Design reviewed against Microsoft's published guidance — what held, what changed, where we differ | Engineering + review board || [docs/roadmap.md](docs/roadmap.md) | Phased delivery plan with exit criteria per phase | Delivery |

## Confirmed design decisions

| Decision | Choice |
|---|---|
| Gateway | Azure API Management, **internal VNet mode** (no public endpoint) |
| Model plane | Azure AI Foundry, **private endpoint only**, multi-region backend pool + circuit breaker |
| Client auth | **Dual** — APIM subscription key for IDE/BYOK, Entra JWT / managed identity for SDK agents |
| Consumer unit | **Per developer** for BYOK, **per workload** for SDK agents |
| Clouds | **Azure Government and Commercial**, selected by deployment parameter |
| Landing zone | Greenfield; hub peering stays a parameter |
| Scale target | 1000+ developers, many teams |
| IaC | `azd` + Bicep, subscription-scope entry point, every capability behind a `deploy*` flag |

## Prior art this is built on

Three working solutions were mined for patterns, policy shapes, and hard-won failure modes:

- **Copilot-CLI-BYOK-Azure** — APIM AI gateway with products/tiers, dual auth, Gov + Commercial
  parameterization, backend pools, self-serve onboarding, VS Code Custom Endpoint integration.
- **Foundry-Private** — private Foundry, three-layer cost control, gateway content safety,
  evaluation runner, agent patterns.
- **health-wellness-agent** — keyless client auth, two-gate content safety, deterministic
  orchestrator with sole tool authority.

The traps those projects hit are captured in
[docs/architecture.md → Known traps](docs/architecture.md#known-traps-carried-forward). Read that
section before writing any policy or Bicep.
The design has also been reviewed against Microsoft's current published guidance — the Azure
Architecture Center gateway guides, the API Management AI gateway documentation, Foundry network
isolation, and the Well-Architected Framework AI guidance. See
[docs/microsoft-guidance-alignment.md](docs/microsoft-guidance-alignment.md) for what changed as a
result.