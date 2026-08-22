# AI Production Operations Playbook

> Operational Playbook — release, on-call, and capacity procedures for Enterprise AI workloads, complementing the [Cost and Model-Routing Playbook](COST_AND_MODEL_ROUTING_PLAYBOOK.md) rather than duplicating it.

---

## Purpose

The Cost and Model-Routing Playbook governs recurring decisions about which model a workload uses and what it costs. This playbook governs the recurring operational activities around getting a change into production safely and keeping it running: release procedure, on-call response, and capacity management. Where the two overlap (e.g., a routing policy change is itself a release), this playbook's release procedure applies to that change, and the Cost and Model-Routing Playbook's Routing Policy Change procedure remains the authority on what the change must satisfy before it is proposed.

## Roles

| Role | Responsibility |
|---|---|
| Release Owner | Executes the staged rollout for a specific model, prompt, or policy version change; accountable for the release succeeding or being rolled back |
| On-Call Engineer | First responder to alerts (per the [Observability Pattern](../engineering-patterns/ENTERPRISE_AI_OBSERVABILITY_PATTERN.md)'s alerting thresholds); the same role already referenced in the [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md) |
| Capacity Owner | Monitors aggregate load against provider rate limits and infrastructure capacity; distinct from the Cost Steward, who monitors spend rather than throughput |

## Recurring Procedures

### 1. Staged Release

**Trigger:** a new model version, prompt version, or routing policy version has passed the [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)'s promotion gates and is ready for production.

**Steps:**

1. Release Owner deploys the new version to a limited traffic percentage (canary), not full production traffic immediately — even a version that passed offline benchmarks per the Evaluation Strategy has not yet been observed against live traffic.
2. Canary traffic is monitored against the same acceptance thresholds used for promotion (per the Evaluation Strategy's Acceptance Criteria), using the trace-level reconstruction from the [Observability Pattern](../engineering-patterns/ENTERPRISE_AI_OBSERVABILITY_PATTERN.md) to compare canary and baseline behavior on equivalent traffic.
3. If canary metrics hold within threshold for a defined minimum observation window, traffic is increased in stages (e.g., 5% → 25% → 100%), not a single jump from canary to full traffic.
4. If canary metrics regress at any stage, the release is rolled back per the [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md)'s Rollback Procedure, and the regression is investigated before the release is retried.

**Output:** a release record — version, stages passed, final traffic percentage, and (if applicable) the rollback trigger and outcome.

### 2. On-Call Rotation and Handoff

**Trigger:** fixed rotation schedule.

**Steps:**

1. On-Call Engineer for the outgoing shift briefs the incoming engineer on any open incidents, active canary releases, or degraded conditions (per the Incident Playbook's incident classes) — a handoff with no briefing is treated as an incomplete handoff.
2. Incoming On-Call Engineer confirms access to the dashboards defined in the Observability Pattern's dashboard composition (routing/orchestration, tool authorization, retrieval, governance, and the cross-cutting trace view) before the outgoing engineer disengages.

**Output:** a recorded handoff with open items explicitly listed, not assumed carried over silently.

### 3. Capacity and Rate-Limit Monitoring

**Trigger:** fixed daily schedule, plus real-time alerting on rate-limit or latency-driven degradation.

**Steps:**

1. Capacity Owner reviews aggregate request volume per model profile against each provider's rate limits and the platform's own infrastructure capacity.
2. A workload trending toward a provider's rate limit is flagged before it is throttled in production, not after — this is a capacity concern distinct from the Cost and Model-Routing Playbook's Weekly Cost Review, which tracks spend rather than throughput headroom.
3. Provider health status (per EDR-0002's Policy Evaluation) is cross-checked against capacity data to distinguish a rate-limit-driven degradation from a provider-health-driven one, since they call for different responses.

**Output:** a capacity status record; workloads approaching a rate-limit threshold are logged with a projected time-to-limit, not only flagged once the limit is reached.

## Escalation Paths

| Situation | Escalate To | Response Target |
|---|---|---|
| Canary metrics regress during staged release | Release Owner executes rollback immediately, per the Incident Playbook's Rollback Procedure | Immediate — canary regression does not wait for a scheduled review |
| On-call handoff occurs with unresolved open incidents and no briefing | Incoming On-Call Engineer escalates to the outgoing engineer's manager before accepting the rotation | Before the rotation is considered handed off |
| A workload's projected time-to-rate-limit falls below a defined buffer | Capacity Owner → Routing Policy Owner (per the [Cost and Model-Routing Playbook](COST_AND_MODEL_ROUTING_PLAYBOOK.md)), to consider a profile change or provider diversification | Before the limit is actually reached |

## Anti-Patterns

- Promoting a version directly to full production traffic because it passed offline evaluation, skipping the staged canary — offline benchmarks and live traffic behavior are not guaranteed to match, which is the entire reason Staged Release exists as a separate step from promotion.
- An on-call handoff that is a calendar event with no actual briefing, leaving the incoming engineer to discover open issues from alerts rather than from the outgoing engineer.
- Treating capacity monitoring as the Cost Steward's job by default — cost and throughput are different failure modes (one is a budget problem, the other is an availability problem) and conflating them delays the correct response to either.

## Related Decisions

- [Cost and Model-Routing Playbook](COST_AND_MODEL_ROUTING_PLAYBOOK.md)
- [Enterprise AI Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)
- [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md)
- [Enterprise AI Observability Pattern](../engineering-patterns/ENTERPRISE_AI_OBSERVABILITY_PATTERN.md)
