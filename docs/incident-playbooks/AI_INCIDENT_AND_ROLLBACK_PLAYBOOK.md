# AI Incident and Rollback Playbook

> Incident Playbook — how to detect, mitigate, recover from, and prevent recurrence of production failures in an Enterprise AI system built on the [Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md) and [EDR-0002](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md).

---

## Scope

This playbook covers incident classes specific to AI systems — model-behavior failures, routing failures, and evaluation-related regressions — not general infrastructure incidents (those follow the organization's standard incident process, which this playbook assumes exists and complements).

## Incident Classes

Each class below maps to a failure mode already documented in [EDR-0002](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md) or the [Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md#failure-paths) — this playbook is the operational response to those documented risks, not a separate taxonomy.

### 1. Model Quality Regression

A model version behaves worse than its accepted baseline (see EDR-0002's "Quality Regression" failure mode).

**Detection:**

- Regression Suite score drop on a model-version change (per [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md));
- online sampled evaluation trending below the accepted threshold;
- spike in user-reported dissatisfaction or correction rate for a specific workload.

**Mitigation (immediate):**

1. On-Call Engineer pins routing policy to the last known-good model version for the affected profile — see Rollback Procedure below.
2. Confirm the pin took effect via routing telemetry (selected model/version field per request).

**Recovery:**

3. Model Evaluation Lead re-runs the offline benchmark against the regressed version to quantify the delta.
4. Provider is notified if the regression traces to a provider-side model update outside the organization's control.

**Prevention:**

- require a promotion gate (per Evaluation Strategy Acceptance Criteria) before any model version serves production traffic, including provider-side "automatic" updates — auto-updating a model in place is treated as a version change requiring re-evaluation, not a no-op.

### 2. Routing Layer Unavailability

The routing layer described in EDR-0002 cannot serve requests (see its "Router Unavailability" failure mode).

**Detection:**

- health check failures on the routing service;
- applications reporting inability to obtain a model selection;
- error-rate spike specifically on the Interaction → Orchestration boundary.

**Mitigation (immediate):**

1. Confirm whether the redundant routing deployment (required by EDR-0002 mitigation) is also affected. If only one instance is down, traffic should already be failing over — verify, do not assume.
2. If the routing layer is fully unavailable, activate the bounded fallback behavior defined for each affected workload. A workload with no approved bounded fallback fails closed — it does not silently call a provider directly, which would violate the Reference Architecture's Orchestration → Enterprise Services contract.

**Recovery:**

3. Restore the routing layer from its last known-good policy snapshot (cached per EDR-0002 mitigation).
4. Verify a sample of routed requests against the restored policy before declaring recovery.

**Prevention:**

- policy snapshots are cached and their freshness is monitored — a stale snapshot silently becoming the recovery target is itself a failure to catch during the retrospective.

### 3. Cost Runaway

Traffic routes to premium model profiles beyond approved budget (see EDR-0002's "Cost Runaway" failure mode).

**Detection:**

- Weekly Cost Review (per the [Cost and Model-Routing Playbook](../operational-playbooks/COST_AND_MODEL_ROUTING_PLAYBOOK.md)) flags a workload trending over budget;
- real-time cost-rate alerting if the overrun is acute enough to require same-day response.

**Mitigation (immediate):**

1. Cost Steward and Routing Policy Owner jointly decide: cap the workload's spend (may degrade to a lower profile) or approve an emergency budget increase. This is never a unilateral engineering decision.

**Recovery:**

2. Apply the decided policy change as a new versioned policy (per the Cost and Model-Routing Playbook's Routing Policy Change procedure) — not a manual override.

**Prevention:**

- workload budgets are enforced with usage limits at the routing layer, not only monitored after the fact.

### 4. Silent Fallback Degradation

A fallback model serves materially lower-quality results without visibility (see EDR-0002's corresponding failure mode).

**Detection:**

- fallback-quality delta metric (per Evaluation Strategy) exceeds the declared acceptable gap for the workload;
- fallback rate spike with no corresponding quality-delta alert (a gap in instrumentation, not an absence of the problem).

**Mitigation (immediate):**

1. Confirm the workload has an approved degraded mode. If it does not, escalate as a policy gap, not just an operational incident — EDR-0002 prohibits fallback for workloads without one.

**Recovery:**

2. Restore primary routing once the underlying cause (see Incident Classes 1–3) is resolved.

**Prevention:**

- fallback events are always exposed in telemetry (per EDR-0002 Observability Requirements); an unmonitored fallback path is treated as a gap to close, not an acceptable state.

### 5. Data Sensitivity or Cross-Layer Leakage

Sensitive data reaches a layer or provider not authorized for it (see the Reference Architecture's "Silent Cross-Layer Data Leakage" failure path and EDR-0002's Security Requirements).

**Detection:**

- Foundation Layer output-filtering sampling (per the Reference Architecture) flags a policy-violating response;
- data residency or retention audit finds a mismatch between declared workload sensitivity and actual provider handling.

**Mitigation (immediate):**

1. Treat as a security incident, not a quality incident — escalate per the organization's standard security incident process in addition to this playbook.
2. Disable routing to the implicated provider or profile for the affected workload's sensitivity class until the leakage path is closed.

**Recovery:**

3. Confirm the specific boundary that failed (which of the Reference Architecture's guarantees was violated) before restoring routing.

**Prevention:**

- data-sensitivity classification is a mandatory, non-optional field of the normalized inference request per EDR-0002 — a request without it should never reach a provider.

## Rollback Procedure (Model or Policy Version)

This is the concrete procedure referenced by "rollback path" throughout this playbook and the Evaluation Strategy's Acceptance Criteria.

1. Identify the last accepted version (model profile mapping or routing policy) from the versioned history — never assume the immediately-previous version was itself good; check its own acceptance record.
2. Pin routing to that version explicitly, rather than deleting or disabling the regressed version (deletion removes the evidence needed for root-causing).
3. Verify via routing telemetry that new requests are being served by the rolled-back version.
4. Record the rollback: what was rolled back, from which version, to which version, why, and who authorized it.
5. The regressed version remains registered but marked ineligible until it either passes re-evaluation or is formally superseded.

A rollback that cannot be verified through telemetry (step 3) is not considered complete.

## Post-Incident Requirements

Every incident under this playbook produces:

- a timeline (detection time, mitigation time, recovery time);
- the specific EDR-0002 or Reference Architecture guarantee that was violated, if any;
- whether the rollback procedure above was followed, and if not, why;
- at least one prevention action with an owner and due date — an incident without a prevention action is not closed.

## Related Decisions

- [EDR-0002 — Model Routing Strategy](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md)
- [Enterprise AI Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md)
- [Enterprise AI Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)
- [Cost and Model-Routing Playbook](../operational-playbooks/COST_AND_MODEL_ROUTING_PLAYBOOK.md)
