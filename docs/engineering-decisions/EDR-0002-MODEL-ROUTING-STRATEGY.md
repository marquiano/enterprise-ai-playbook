# EDR-0002 — Model Routing Strategy

## Status

Proposed

## Context

Enterprise AI platforms rarely operate efficiently with a single model.

Different workloads have different requirements for:

- reasoning quality;
- latency;
- context window;
- structured output reliability;
- privacy;
- cost;
- availability;
- regional deployment;
- multimodal capabilities.

Using the most capable model for every request maximizes simplicity but creates unnecessary cost, latency and vendor concentration.

Using the cheapest model for every request reduces cost but increases failure rates and limits the complexity of supported workloads.

The platform therefore needs a routing strategy that selects a model based on workload characteristics and operational policy.

## Decision Drivers

The routing strategy must:

- preserve minimum quality requirements;
- control inference cost;
- support multiple model providers;
- avoid direct provider dependencies in business services;
- allow policy changes without application rewrites;
- produce auditable routing decisions;
- provide deterministic fallbacks;
- prevent silent quality degradation.

## Decision

Adopt a policy-based model routing layer between AI orchestration and model providers.

Applications submit a normalized inference request containing:

- task type;
- risk classification;
- latency target;
- quality threshold;
- context size;
- data sensitivity;
- required capabilities;
- budget class.

The routing layer evaluates these attributes against versioned policies and selects an approved model profile.

Business applications must not call model providers directly.

## Model Profiles

Models are registered as capability profiles rather than referenced directly by provider-specific identifiers.

Example profiles:

| Profile | Intended Workload |
|---|---|
| Economy | Classification, extraction and low-risk transformation |
| Standard | General enterprise workflows and grounded generation |
| Reasoning | Complex analysis, planning and high-ambiguity tasks |
| Private | Sensitive workloads requiring controlled deployment |
| Multimodal | Image, document, audio or video processing |

A profile may map to different providers or model versions over time.

## Routing Flow

```mermaid
flowchart LR

A["Application"] --> B["Normalized AI Request"]
B --> C["Policy Evaluation"]
C --> D["Capability Registry"]
D --> E["Model Selection"]
E --> F["Provider Adapter"]
F --> G["Model Provider"]

G --> H["Evaluation and Telemetry"]
H --> C
Policy Evaluation

Routing policies may consider:

Mandatory capability compatibility.
Data sensitivity and deployment restrictions.
Minimum quality threshold.
Maximum acceptable latency.
Available budget.
Provider health and regional availability.
Historical evaluation performance.
Current rate limits and capacity.

Hard constraints must be evaluated before optimization criteria.

A model that violates a security or quality constraint must never be selected solely because it is cheaper.

Alternatives Considered
Alternative A — Single Strategic Model

All workloads use one enterprise-approved model.

Advantages
simple integration;
fewer provider contracts;
consistent behavior;
reduced operational surface.
Rejected Because
cost is inefficient across heterogeneous workloads;
provider outages affect the entire platform;
specialized workloads remain underserved;
migration becomes expensive;
vendor concentration risk increases.
Alternative B — Application-Owned Model Selection

Each application team selects and integrates its own models.

Advantages
local autonomy;
rapid experimentation;
workload-specific optimization.
Rejected Because
duplicated integrations;
inconsistent governance;
weak cost control;
limited auditability;
fragmented evaluation;
direct vendor coupling.
Alternative C — Cheapest-Available Routing

The router always selects the lowest-cost compatible model.

Advantages
aggressive cost reduction;
simple optimization objective.
Rejected Because
compatibility does not guarantee acceptable quality;
model regressions may remain undetected;
high-risk workloads require stronger controls;
cost becomes the dominant architectural concern.
Consequences
Positive
provider independence improves;
routing policy can evolve centrally;
application teams use a stable abstraction;
cost and performance can be optimized per workload;
routing decisions become auditable;
fallback behavior can be standardized.
Negative
the routing layer becomes a critical platform dependency;
policy design introduces operational complexity;
model profiles require continuous evaluation;
abstraction may hide provider-specific capabilities;
incorrect routing policies can cause systemic quality failures.
Failure Modes
Incorrect Capability Metadata

A model may be selected for a capability it does not reliably support.

Mitigation:

validate profiles through automated evaluation;
version capability declarations;
prevent self-declared production eligibility.
Quality Regression

A provider may update a model without preserving observed behavior.

Mitigation:

maintain benchmark suites;
monitor model-version changes;
require promotion gates before policy activation.
Router Unavailability

Applications may be unable to select a provider.

Mitigation:

deploy the router redundantly;
cache approved policy snapshots;
define bounded fallback behavior.
Cost Runaway

Policies may route excessive traffic to premium models.

Mitigation:

define workload budgets;
monitor cost by application and business unit;
enforce usage limits and escalation rules.
Silent Fallback Degradation

A fallback model may produce materially lower-quality results.

Mitigation:

declare fallback quality classes;
expose fallback events in telemetry;
prohibit fallback for workloads without an approved degraded mode.
Observability Requirements

Each routing decision must record:

request identifier;
workload;
selected model profile;
provider and model version;
policy version;
decision factors;
fallback status;
latency;
token or compute usage;
estimated cost;
evaluation outcome when available.

Sensitive input and output data must not be logged by default.

Security Requirements

Routing must enforce:

approved provider lists;
data residency requirements;
workload sensitivity classifications;
provider-specific data retention restrictions;
encryption and credential isolation;
policy-based denial when no compliant model is available.
Acceptance Criteria

The decision is considered successfully implemented when:

applications use a provider-neutral request contract;
direct model-provider calls are blocked or explicitly exempted;
routing policies are versioned;
every decision is auditable;
fallback behavior is documented and tested;
evaluation results influence model eligibility;
cost and latency are measurable by workload;
no model can bypass security constraints.
Reversal Criteria

This decision should be reconsidered if:

the organization operates only one stable workload;
regulatory policy permits only a single provider or deployment;
routing complexity exceeds the value generated;
evaluation data shows no meaningful benefit from differentiated profiles;
the routing layer becomes an unacceptable reliability bottleneck.
Related Decisions
EDR-0001 — Playbook Scope
Review Notes

This decision must be validated against at least one runnable implementation or worked case study before being marked Accepte