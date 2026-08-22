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

### 6. Prompt Injection Exploited

A manipulation embedded in processed content (retrieved document, tool output, third-party content) successfully steered the model into a policy-violating output or an unauthorized tool-call proposal, despite the [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md) pattern.

**Detection:**

- injection/jailbreak resistance rate (per [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)) drops below threshold;
- a policy-violating output or unauthorized tool-call proposal is traced to manipulated content in the request context.

**Mitigation (immediate):**

1. Confirm the [EDR-0003](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) authorization boundary held — i.e., that any resulting tool-call proposal was rejected, not executed. If it was executed, this is also incident class 7 below.
2. If the manipulation source is a specific content provider (a document source, a third-party API), suspend ingestion from that source pending review.

**Recovery:**

3. Add the successful manipulation to the adversarial reference set (per the pattern's Benefit of versioned test material) so the same bypass class is caught by the offline benchmark before the next release.

**Prevention:**

- adversarial evaluation is a release gate, not a one-time audit — every discovered bypass strengthens the gate for the next version, per the pattern's methodology.

### 7. Agent Tool-Scope Violation or Data Exfiltration

A tool call executed with a scope broader than the requesting task justified, or data reached a destination the task did not authorize — the [EDR-0003](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) permission boundary was bypassed, misconfigured, or itself misclassified the operation's risk tier.

**Detection:**

- audit comparison of granted versus actually-used scope (per EDR-0003 Observability Requirements) flags an over-grant or an execution outside declared scope;
- an irreversible-or-external-effect action is found to have executed without a recorded human approval.

**Mitigation (immediate):**

1. Revoke the implicated task's active grants and suspend the agent session.
2. If data left an authorized boundary, treat this as a security incident in addition to this playbook — same escalation posture as [Incident Class 5 (Data Sensitivity or Cross-Layer Leakage)](#5-data-sensitivity-or-cross-layer-leakage).

**Recovery:**

3. Reconstruct the authorization trail per EDR-0003's Acceptance Criteria — task, scope, and any approver — to determine whether the boundary failed to enforce, or was correctly enforced against a misclassified risk tier.

**Prevention:**

- default classification for any new or unclassified tool is the highest risk tier, per EDR-0003's mitigation for this exact failure mode — an incident here often traces back to a tool that shipped without a completed risk classification.

### 8. Runaway Agent Execution or Goal Drift

An agentic workload's planning loop drifts from the original task intent or gets stuck without making progress — see the [Enterprise Agent Architecture](../reference-architectures/ENTERPRISE_AGENT_ARCHITECTURE.md)'s "Goal Drift Across Steps" and "Runaway Execution Loop" failure paths for the underlying mechanism.

**Detection:**

- plan-intent drift rate or task-completion-within-budget metric (per [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)) exceeds threshold;
- step count or resource usage for a running task exceeds its declared budget without reaching completion.

**Mitigation (immediate):**

1. Terminate the task at its budget boundary — this should already be automatic per the Agent Architecture's Planner → Task Completion contract; if manual intervention was required, that contract failed to enforce and is itself part of the incident.
2. If drift is traced to a specific intermediate step, treat as Incident Class 6 (Prompt Injection Exploited) in addition to this class if injected content is implicated.

**Recovery:**

3. Report the task as incomplete rather than silently retrying without limit; a human reviews whether the partial work (if any) is usable or must be discarded.

**Prevention:**

- every agentic task has an explicit, non-optional step and resource budget declared before execution begins — an agent without one is not eligible for production traffic, mirroring the Evaluation Strategy's "no threshold defined" is not a valid state" rule.

### 9. Ungrounded Generation or Hallucination via Retrieval Failure

A retrieval-backed workload asserts a claim without valid supporting citation, or cites a passage that does not actually support the claim — a bypass of [EDR-0004](../engineering-decisions/EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md)'s grounding enforcement, or a symptom of the [Enterprise RAG Architecture](../reference-architectures/ENTERPRISE_RAG_ARCHITECTURE.md)'s "Index Staleness" or "Silent Ungrounded Generation" failure paths.

**Detection:**

- citation support rate (per [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)) drops below threshold, or diverges from citation coverage rate;
- passage freshness at time of use falls outside the workload's declared staleness target;
- user-reported factual error traced to a cited or missing citation.

**Mitigation (immediate):**

1. Confirm whether the affected claims carry citations at all. No citation with no refusal indicates the generation-time enforcement mechanism itself failed — treat as a defect in the enforcement path, not merely a content-quality issue.
2. If citations are present but unsupported, check passage freshness first — a stale-but-once-correct source is a different root cause than a genuinely irrelevant retrieval.

**Recovery:**

3. If traced to index staleness, trigger re-ingestion for the affected source and verify the corrected passage is retrievable before considering the incident resolved.
4. If traced to an enforcement gap, follow the [Rollback Procedure](#rollback-procedure-model-or-policy-version) below for the generation policy version, consistent with any other regression.

**Prevention:**

- citation support rate and citation coverage rate are reviewed together on a fixed schedule (per the [Cost and Model-Routing Playbook](../operational-playbooks/COST_AND_MODEL_ROUTING_PLAYBOOK.md)'s operating-cadence pattern), not only when a user reports an error;
- freshness targets are declared per workload before launch, per EDR-0004's Acceptance Criteria — a workload with no declared staleness target has an unbounded exposure to this incident class.

### 10. Unapproved or Unclassified AI Use-Case in Production

A use case is found operating with production model routing or tool authorization but no recorded risk classification, no confirmed accountable owner, or (for high-risk use cases) no recorded approval — a bypass or gap in [EDR-0005](../engineering-decisions/EDR-0005-USE-CASE-RISK-CLASSIFICATION-AND-APPROVAL-GATE.md)'s gate, matching the Reference Architecture's "Unclassified or Unapproved Use-Case Reaches Production" failure path.

**Detection:**

- classification coverage rate or accountable-owner currency rate (per [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)) falls below 100%;
- the [AI Governance Review Playbook](../operational-playbooks/AI_GOVERNANCE_REVIEW_PLAYBOOK.md)'s periodic reclassification review finds active traffic from an unregistered use-case identity.

**Mitigation (immediate):**

1. Suspend the use case's model routing and tool authorization pending classification — per EDR-0005, an unclassified or unowned use case is treated as ineligible for production access, not grandfathered in until reviewed.
2. Identify how it obtained access without registration; if through a path that bypasses the routing/authorization layers' registration check, this is also an architecture defect, not only a process gap.

**Recovery:**

3. Route the use case through Use-Case Intake and Classification (per the Governance Review Playbook) before restoring access.
4. If found high-risk, require the recorded approval before restoration, per the same procedure's High-Risk Approval step.

**Prevention:**

- registration checks at the routing and tool-authorization layers are structural, not advisory — an unregistered use case should have no path to either, per EDR-0005's Acceptance Criteria; an incident here often traces back to a path that was never wired through those checks.

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
- [EDR-0003 — Agent Tool-Scope and Permission Boundary](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md)
- [Enterprise AI Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md)
- [Enterprise AI Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)
- [Cost and Model-Routing Playbook](../operational-playbooks/COST_AND_MODEL_ROUTING_PLAYBOOK.md)
- [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md)
- [Enterprise Agent Architecture](../reference-architectures/ENTERPRISE_AGENT_ARCHITECTURE.md)
- [EDR-0004 — Retrieval Grounding and Citation Enforcement](../engineering-decisions/EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md)
- [Enterprise RAG Architecture](../reference-architectures/ENTERPRISE_RAG_ARCHITECTURE.md)
- [EDR-0005 — AI Use-Case Risk Classification and Approval Gate](../engineering-decisions/EDR-0005-USE-CASE-RISK-CLASSIFICATION-AND-APPROVAL-GATE.md)
- [AI Governance Review Playbook](../operational-playbooks/AI_GOVERNANCE_REVIEW_PLAYBOOK.md)
