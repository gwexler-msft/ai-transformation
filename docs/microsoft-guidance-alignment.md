# Alignment with Microsoft's published guidance

**BLUF.** The design was reviewed against Microsoft's current public guidance for AI gateways,
API Management, Foundry network isolation, and the Well-Architected Framework. Most of it holds
up — the core shape is exactly what Microsoft recommends. Six things changed, three of them
materially. This document records what was checked, what moved, and where we knowingly differ.

## Sources reviewed

| Source | What it covers |
|---|---|
| [AI gateway capabilities in API Management](https://learn.microsoft.com/azure/api-management/genai-gateway-capabilities) | The AI gateway feature set — token limits, semantic caching, content safety, load balancing, unified model API, MCP and A2A support |
| [Access Foundry models through a gateway](https://learn.microsoft.com/azure/architecture/ai-ml/guide/azure-openai-gateway-guide) | Why a gateway, evaluated against the five WAF pillars, and when *not* to build one |
| [Use a gateway in front of Foundry deployments or instances](https://learn.microsoft.com/azure/architecture/ai-ml/guide/azure-openai-gateway-multi-backend) | The four multi-backend topologies, failover, quota scoping, data sovereignty |
| [API Management with a virtual network](https://learn.microsoft.com/azure/api-management/virtual-network-concepts) | Which tiers support VNet injection, internal vs external mode, private endpoints |
| [Create and manage a unified model API](https://learn.microsoft.com/azure/api-management/unified-model-api) | Client-facing model aliases, `/models` discovery, cross-provider format translation |
| [Set up logging for language model APIs](https://learn.microsoft.com/azure/api-management/api-management-howto-llm-logs) | Built-in LLM logging, `ApiManagementGatewayLlmLog`, analytics dashboard |
| [Configure network isolation for Foundry](https://learn.microsoft.com/azure/ai-foundry/how-to/configure-private-link) | Inbound and outbound isolation, VNet injection, agent tool support matrix, limitations |
| [Configure virtual networks for Foundry Tools](https://learn.microsoft.com/azure/ai-services/cognitive-services-virtual-networks) | Network ACLs, trusted-services bypass, private endpoint behaviour |
| [WAF — application design for AI workloads](https://learn.microsoft.com/azure/well-architected/ai/application-design) | Layered architecture, gateway policy enforcement, caching, model deprecation |

---

## What was already right

No change needed on these. Worth stating explicitly, because they were the contentious decisions.

| Our decision | Microsoft's position |
|---|---|
| APIM classic Developer/Premium, internal VNet injection | Confirmed — classic Developer and Premium are the tiers that support VNet injection. Internal mode is the documented pattern for private-only gateway access over ExpressRoute or VPN. |
| Terminate the client credential at the gateway, reauthenticate with managed identity | "Strongly consider implementing credential termination and reestablishment. The client is authorized at the gateway, and then the gateway is authorized via Azure RBAC to that instance." |
| Foundry reachable only through the gateway | "Deploy the gateway into a virtual network that contains a subnet for the instance's Private Link private endpoint. Apply NSG rules to that subnet to allow only the gateway access. Disallow all other data plane access." |
| Circuit breaker honouring `Retry-After` | Called out specifically: don't keep hitting an endpoint returning 429, break the circuit, and use the `Retry-After` value as authoritative. |
| Logical model aliases decoupled from deployments | Now a product feature — see the unified model API below. Also a WAF recommendation: "abstract models behind consistent interfaces to enable swapping or upgrading without application changes." |
| Gateway as the chargeback point | "Chargeback management is typically handled by an API gateway" — and a gateway is the only place client-level usage can be distinguished when clients share an instance. |
| Enforce at the gateway, not in the client | "Don't rely solely on SDK retries and timeouts. Enforce limits, authentication, quotas, and timeouts at the gateway and policy layer." |
| Token limits keyed on subscription | The documented `llm-token-limit` pattern, `counter-key="@(context.Subscription.Id)"`. |
| Content safety at the gateway | Documented `llm-content-safety` policy against a private Content Safety account. |

---

## What changed

### 1. Anthropic gets no native route — and that is a tier problem, not a preference

**Was:** an `/anthropic/v1/messages` API exposing the native Anthropic Messages schema.

**Problem:** the native Anthropic Messages schema is supported only in the API Management **v2
tiers**. We are on classic, because classic is the only thing that does VNet injection in
Government. Those two constraints are mutually exclusive.

**Now:** Anthropic models are exposed through the **unified model API**, which takes OpenAI Chat
Completions from the client and translates to the Anthropic format at the backend. Clients see one
shape. This is arguably better — no client needs to learn a second request format — but it was not
a free choice.

### 2. The alias layer is a product feature, not something we build

**Was:** hand-rolled alias resolution in policy, plus a custom `/models` endpoint.

**Now:** APIM's **unified model API** does exactly this — client-facing aliases mapped to backend
deployments, a `/models` discovery endpoint, cross-provider format translation, and failover
across providers. It is in preview, and it reaches classic tiers through the **AI Gateway early
release channel**.

We keep the hand-rolled design documented as the fallback, because preview features in Government
need verification before anyone commits to them. But building it from scratch when Microsoft ships
it is the wrong default. Availability is now open item **O9**.

### 3. Multi-region pools do not give us more quota

**Was:** an implication that a multi-region backend pool helps with capacity for 1000+ developers.

**Problem:** "Standard quotas are subscription level, not instance level. Load balancing against
standard instances in the same subscription doesn't achieve additional throughput." And more
bluntly: "Don't implement a unified gateway solely to increase quota."

**Now:** the pool is for **availability**, full stop. Capacity comes from **deployment type** —
Global Standard draws on Azure's global capacity and is the intended answer to throttling. We had
no mention of deployment types at all, which was the biggest single omission in the original draft.

This turns into a compliance question immediately, which is the next item.

### 4. Deployment type is a data-residency decision before it is a capacity decision

Global Standard keeps data **at rest** in the geography but may process inference in **any**
hosting location. Data Zone constrains processing to a defined zone. Regional Standard pins both.

For a Government deployment this is likely to be the constraint that sets the capacity ceiling.
It is now open item **O8** and it is the question I would want answered earliest, because it
changes the sizing model, the cost model, and possibly the region strategy.

### 5. The Responses API is stateful, and that conflicts with pooling

**Problem:** "Stateful interactions, such as the Responses API, require the gateway to maintain
session affinity so that subsequent requests from the same session are routed to the same back-end
instance."

We expose `/responses` (VS Code uses it) *and* we pool across regions. Round-robin breaks
conversations mid-flight.

**Now:** session-aware pool distribution for that route — APIM supports it. Microsoft also
suggests returning a 429 to a pinned client rather than silently rerouting it to a backend with no
history, which is the honest failure mode. Added as decision **D11** and open item **O11**.

### 6. Built-in LLM logging replaces a chunk of custom telemetry

**Was:** custom `emit-metric` policies and hand-written KQL for token capture.

**Now:** API Management has a dedicated generative-AI diagnostic writing to
`ApiManagementGatewayLlmLog` — prompt, completion and total tokens plus model name, per request,
with no policy code — and a built-in *Language models* analytics dashboard over it. Custom
dimensions still matter for consumer attribution, but the token capture is a platform feature.

Two caveats came with it, both now recorded as traps:

- Token usage **can be missing or inaccurate when a stream breaks**. Anything billed on token
  counts needs reconciliation against actual Azure spend.
- Prompt and completion **content capture is available** as a separate opt-in (32 KB per entry,
  2 MB ceiling). Still off by default in our design, but it is the mechanism if audit requirements
  later demand full interaction records — and those logs export directly as a Foundry evaluation
  dataset.

---

## Smaller corrections folded in

| Item | Change |
|---|---|
| Private endpoint hostname | Clients must call the account's custom subdomain, never `*.privatelink.*`. Added as trap T14. |
| Private endpoint subnet | NSG permitting only the gateway subnet is now explicit in the network design, not implied. Trap T15. |
| Model version consistency | Every pool member must serve the same model *and version*. Never fail over X to X+1. Trap T16. |
| Throttle prediction | Don't pre-empt 429s by tracking prior consumption — drive routing from response codes. This is the argument against `estimate-prompt-tokens`. |
| Health checks | APIM's default `/status-...` endpoint reports the gateway only. A dedicated health API is needed for backend visibility. |
| Gateway as single point of failure | A single-region gateway fronting multi-region backends protects against model outages, not gateway outages. Stated plainly; open item O10. |
| Resource-group layout | Gateway in its own resource group, colocated with the model region. Decision D12. |
| Azure Policy | Use built-in AI-services policies to stop drift between instances, which silently breaks failover. |
| Semantic caching | Named as the largest unused cost lever, with the cache-key governance rules that make it safe. New section 10.1. |
| Foundry outbound networking | Cannot be added or changed after deployment. Trap T21. |
| Reserved CIDR | `172.17.0.0/16` collides with Docker bridge networking. Trap T22. |
| Trusted-services bypass | `networkAcls.bypass = AzureServices` exists and is a deliberate hole, not a default. |

---

## Where we deliberately differ

| Microsoft says | We do | Why |
|---|---|---|
| Foundry has a built-in AI gateway you can enable from the portal | We use a standalone API Management instance | The embedded gateway provisions **public**, and it fronts a **single** Foundry resource. Neither works for a private, multi-account, multi-region design. Trap T20. |
| Global Standard deployments solve throttling without a gateway | We still build the gateway | Capacity was never the reason. Identity, allowlisting, per-consumer quotas, attribution and audit are, and none of them come from a deployment type. |
| Consider whether the gateway's cost and complexity is justified | It is, at this scale | Microsoft is right to ask. At 1000+ developers with no per-client attribution and no shared control point, the answer is not close. At ten users it would be. |
| Semantic caching gives large cost savings | Deferred past v1 | The cache-key rules are subtle and the failure mode is cross-user data leakage. Build it when there is measured traffic to justify it. |
| A gateway can add session affinity for stateful APIs | We do, but we also question whether Responses belongs in v1 | Session affinity constrains everything else about pooling. Worth knowing whether we need it before designing around it. |

---

## Things we are not doing yet that are worth a second look

- **Azure API Center.** The product for an organizational catalog of APIs, MCP servers and agents,
  with a self-service developer portal. That is most of governance area 1 and part of area 2.
  Currently we plan a custom onboarding portal — API Center may replace or feed it.
- **Application Gateway with WAF in front of internal APIM.** A documented pattern that could give
  developer laptops a controlled, WAF-inspected entry point without VPN or VDI. Directly relevant
  to the access-path problem in open item O3.
- **A2A agent APIs and MCP governance in the Foundry control plane.** Foundry can now register
  agents and MCP tools for centralized inventory and policy. Relevant to governance area 10 when
  we get there.
- **Multi-region API Management.** One control plane, replicated regional gateways, built-in
  latency-based FQDN routing. The answer if the availability target tightens.
