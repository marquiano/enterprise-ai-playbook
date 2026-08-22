# Engineering Pattern — Enterprise AI Observability

> Engineering Pattern — a single, cross-component instrumentation pattern that unifies the Observability Requirements already decided separately in EDR-0002, EDR-0003, EDR-0004, and EDR-0005.

---

## Context

Four EDRs each declared their own Observability Requirements section, correctly, because each decision needed its own record: [EDR-0002](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md) requires routing-decision records, [EDR-0003](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) requires tool-authorization records, [EDR-0004](../engineering-decisions/EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md) requires grounding/citation records, and [EDR-0005](../engineering-decisions/EDR-0005-USE-CASE-RISK-CLASSIFICATION-AND-APPROVAL-GATE.md) requires governance records.

Implemented independently, these four requirement sets produce four disconnected logs. A single user request that routes through model selection, invokes a tool, retrieves grounding context, and belongs to a governed use case generates four separate records with no shared identifier — reconstructing what actually happened for that one request means manually correlating timestamps across four systems, which does not scale and is exactly how root-causing an incident becomes slow. This pattern does not add new requirements; it gives the requirements already decided a shared structure so they compose into one traceable picture instead of four disconnected ones.

## Applicability

Apply this pattern to any workload that touches more than one of the four decisions above in a single request — which, in practice, is most production Enterprise AI workloads, since a routed model call (EDR-0002) is the near-universal entry point and the others layer on top of it. A workload touching only one of the four decisions still benefits from adopting the shared schema below, since it costs nothing extra and keeps the workload consistent with the rest of the platform as it grows.

## The Pattern

### 1. A Single Trace Identifier Per Request

Every request is assigned one trace identifier at the point it enters AI Orchestration (per the [Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md)'s Interaction → AI Orchestration contract), and that identifier is propagated, unchanged, through every subsequent record: the routing decision (EDR-0002), any tool authorization (EDR-0003), any retrieval and citation (EDR-0004), and the governance identity of the use case it belongs to (EDR-0005). A record from any of the four decisions that lacks this trace identifier cannot be correlated with the request it belongs to, which defeats the purpose of collecting it.

### 2. A Shared Event Schema

Each of the four decisions already specifies what its own record must contain (see their respective Observability Requirements sections). This pattern adds one shared envelope every record carries in addition to its decision-specific fields:

- trace identifier (per Control 1);
- use-case identity (per EDR-0005's governance record — every event is attributable to a registered, classified use case, never an anonymous or unregistered one);
- event type (routing-decision / tool-authorization / retrieval-citation / governance-check);
- timestamp;
- outcome (success / denied / flagged / refused) using consistent terms across all four event types, so a dashboard or query does not need per-event-type translation logic.

### 3. Trace-Level Reconstruction, Not Just Event-Level Logging

The point of Control 1 and 2 together is that a single query by trace identifier reconstructs the full path of one request across all four decisions: which model was selected and why, what tools were authorized and executed, what was retrieved and cited, and which use case (and its risk classification) the request belonged to. This reconstruction is what an incident responder needs under the [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md), and what a governance reviewer needs under the [AI Governance Review Playbook](../operational-playbooks/AI_GOVERNANCE_REVIEW_PLAYBOOK.md) — both currently have to assemble this manually without a shared trace identifier.

### 4. Alerting Thresholds Map to Evaluation Strategy Metrics, Not to Raw Event Counts

Every metric defined in the [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md) (groundedness, unauthorized tool-call attempt rate, citation support rate, plan-intent drift rate, classification coverage rate, and the rest) has a corresponding alert threshold, derived from that metric's acceptance criteria, not invented separately by whoever configures the alerting system. This keeps "what counts as a problem" defined in one place (the Evaluation Strategy) rather than reimplemented, possibly inconsistently, in dashboard configuration.

### 5. Dashboard Composition Follows the Same Structure as the Reference Architecture's Layers

A dashboard is composed per architectural layer (routing/orchestration, tool authorization, retrieval, governance) with a cross-cutting trace-level view that can drill from any layer's summary into a specific trace's full reconstruction (Control 3). This avoids the common failure of building one undifferentiated dashboard that tries to show everything at once and ends up showing nothing clearly.

## Benefits

- a single trace identifier makes cross-component root-causing a query, not a manual correlation exercise;
- alerting definitions stay synchronized with the Evaluation Strategy's actual acceptance thresholds instead of drifting from them over time;
- the shared envelope makes it possible to answer "show me everything that happened for this use case" or "show me every event tied to this trace" without four separate lookups;
- new decisions added in the future (a hypothetical EDR-0006) adopt the same envelope rather than inventing a fifth disconnected logging scheme.

## Limitations

- this pattern does not reduce what must be logged — it is a structural convention over the four EDRs' existing requirements, not a replacement for reading them;
- trace propagation must be implemented consistently across every component that touches a request; a single component that drops or regenerates the trace identifier breaks reconstruction for every request passing through it;
- sensitive input/output data must still not be logged by default, per EDR-0002's Observability Requirements — this pattern governs structure and correlation, not what content is captured, and does not relax that constraint.

## Related Artifacts

- [EDR-0002 — Model Routing Strategy](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md)
- [EDR-0003 — Agent Tool-Scope and Permission Boundary](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md)
- [EDR-0004 — Retrieval Grounding and Citation Enforcement](../engineering-decisions/EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md)
- [EDR-0005 — AI Use-Case Risk Classification and Approval Gate](../engineering-decisions/EDR-0005-USE-CASE-RISK-CLASSIFICATION-AND-APPROVAL-GATE.md)
- [Enterprise AI Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)
- [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md)
- [AI Governance Review Playbook](../operational-playbooks/AI_GOVERNANCE_REVIEW_PLAYBOOK.md)
