# Cost and Model-Routing Playbook

> Operational Playbook — standardizes the recurring activities that keep [EDR-0002's](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md) routing policy correct, cost-controlled, and current.

---

## Purpose

EDR-0002 decides that model selection is policy-based, not hardcoded. This playbook is the operational procedure that keeps that policy correct over time — routing policy is not a "set once" artifact.

## Roles

| Role | Responsibility |
|---|---|
| Routing Policy Owner | Approves changes to routing policy; sole approver for hard-constraint changes (security, data residency) |
| Cost Steward | Monitors spend by workload and business unit; proposes budget-driven policy adjustments |
| Model Evaluation Lead | Supplies evaluation results (see [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)) that determine model eligibility |
| On-Call Engineer | Executes emergency routing changes (profile disablement, fallback activation) outside the normal review cycle |

A single person may hold multiple roles in a small organization, but the responsibilities remain distinct and each change is still attributable to a role, not a person acting informally.

## Recurring Procedures

### 1. Weekly Cost Review

**Trigger:** fixed weekly schedule.

**Steps:**

1. Cost Steward pulls cost-per-workload and cost-per-model-profile from routing telemetry (per EDR-0002 Observability Requirements).
2. Compare against the workload's declared budget class.
3. Flag any workload trending to exceed its budget class within the current billing period.
4. For flagged workloads, propose either a profile downgrade (e.g. Reasoning → Standard) or a budget class increase — never a silent profile downgrade without the owning team's sign-off, since quality may regress.

**Output:** a dated cost review record listing flagged workloads and the decision made for each.

### 2. Model Eligibility Refresh

**Trigger:** a new evaluation result lands (per the Evaluation Strategy's Regression Suite or Offline Benchmark cadence), or on a fixed monthly schedule, whichever comes first.

**Steps:**

1. Model Evaluation Lead confirms the model version's latest scores meet its profile's acceptance thresholds.
2. If thresholds are not met, the model version is marked ineligible for that profile in the capability registry — it is not removed silently; the change is logged with the failing metric.
3. Routing Policy Owner reviews any newly ineligible model for downstream impact (which workloads route to it, what the fallback is).

**Output:** an updated, versioned capability registry entry.

### 3. Routing Policy Change

**Trigger:** a proposed change to policy weighting, a new model profile, or a new hard constraint.

**Steps:**

1. Proposer documents the change and the decision drivers it addresses (referencing EDR-0002's Decision Drivers where applicable).
2. Change is evaluated against the Acceptance Criteria in EDR-0002 — does it preserve provider neutrality, auditability, and the hard-constraint-before-optimization rule?
3. Routing Policy Owner approves or rejects.
4. Approved changes are deployed as a new policy version, never as a mutation of the active version — the previous version remains available for rollback.

**Output:** a new policy version number, changelog entry, and rollback target.

### 4. Provider Health Check

**Trigger:** fixed daily schedule, plus real-time alerting on provider error-rate spikes.

**Steps:**

1. Confirm each registered provider's health and regional availability status used by Policy Evaluation.
2. Any provider below its availability threshold is marked degraded in the capability registry, triggering fallback routing for affected profiles.
3. On recovery, provider status is restored only after a defined stability window, not immediately on the first healthy check.

**Output:** provider status log; degraded/recovered events are observable in routing telemetry.

## Escalation Paths

| Situation | Escalate To | Response Target |
|---|---|---|
| Workload budget exceeded mid-cycle with no reviewed mitigation | Cost Steward → Routing Policy Owner | Same business day |
| Model profile has zero eligible models (all versions failed evaluation) | Model Evaluation Lead → Routing Policy Owner | Immediate — workload has no compliant route |
| Router itself unavailable | On-Call Engineer, per the [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md) | Immediate |
| Proposed policy change would allow a hard-constraint violation (security, residency) | Any reviewer → Routing Policy Owner, change is blocked | Before merge, not after |

## Anti-Patterns

- Adjusting routing policy directly in a production config without going through a versioned policy change — see EDR-0002's requirement that every routing decision be auditable.
- Treating a cost overrun as purely an engineering problem — budget class is a business decision and belongs with the workload owner, not silently absorbed by downgrading quality.
- Letting "temporary" emergency routing changes made by the On-Call Engineer persist past the incident without a follow-up policy review.

## Related Decisions

- [EDR-0002 — Model Routing Strategy](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md)
- [Enterprise AI Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)
