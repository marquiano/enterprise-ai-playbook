# Enterprise AI Evaluation Strategy

> Evaluation Framework — defines how Enterprise AI systems are measured, gated, and monitored.

---

## Purpose

An AI system that is not evaluated on a defined schedule, against defined criteria, is not operated — it is merely running. This document defines what must be measured, how, and what the results are allowed to gate.

It applies to every model profile registered under [EDR-0002 — Model Routing Strategy](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md): a model profile with no evaluation coverage is not eligible for production routing.

## Evaluation Classes

Evaluation is not a single activity. Four classes cover different risks and run at different cadences.

| Class | Question It Answers | Cadence |
|---|---|---|
| Offline Benchmark | Does this model/prompt version meet quality thresholds before release? | Pre-deployment, on every model or prompt version change |
| Online Sampled Evaluation | Is production behavior still within the thresholds observed offline? | Continuous, sampled |
| Regression Suite | Did a provider-side model update change observed behavior? | On every detected model-version change, and on a fixed schedule (weekly minimum) |
| Human Review | Does the system behave acceptably on judgment calls automated metrics cannot resolve? | Sampled, weighted toward high-risk workload classes |

## Metrics

Metrics are grouped by what they protect against. A workload's required metric set is determined by its risk classification (see [EDR-0002](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md) Decision).

### Task Quality

- task success rate against a labeled reference set;
- structured-output validity rate (schema conformance for workloads requiring structured output);
- groundedness — proportion of claims traceable to supplied context, for any workload using retrieval.

### Safety and Policy Compliance

- policy violation rate (content, data handling, action authorization);
- refusal correctness — the system refuses when it should, and does not refuse when it should not.

### Operational Quality

- latency distribution (p50/p95/p99) against the workload's declared latency target;
- fallback rate and fallback-quality delta versus primary model;
- cost per successful task.

### Stability

- output variance across repeated identical requests, for workloads requiring determinism;
- score delta between consecutive regression-suite runs on the same model version (should be ~0; a nonzero delta on an unchanged model version indicates measurement instability, not model drift).

### Security

- injection/jailbreak resistance rate — proportion of the adversarial reference set (per the [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md) pattern) that does not produce a policy-violating output or an unauthorized tool-call proposal;
- unauthorized tool-call attempt rate — proposed tool calls rejected at the [EDR-0003](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) authorization boundary, tracked separately from calls that were never proposed, since a rejection is the permission boundary working as intended, not itself a failure;
- scope-to-usage ratio — granted scope versus actually-used scope per EDR-0003's audit requirement, flagging systematic over-grants.

Unlike task-quality metrics, a rising unauthorized tool-call attempt rate is not automatically a regression — it may indicate the injection-defense pattern is under active attack while the authorization boundary is correctly holding. Both metrics must be read together, per the Reference Architecture's [failure paths](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md#failure-paths) for this incident class.

### Agentic Workloads

For workloads with a multi-step planning loop, per the [Enterprise Agent Architecture](../reference-architectures/ENTERPRISE_AGENT_ARCHITECTURE.md):

- task completion rate within the declared step and resource budget — a task that succeeds only by exceeding its budget is not a pass;
- plan-intent drift rate — proportion of sampled tasks where a step's declared intent diverged from the original task intent beyond the defined threshold (per the Agent Architecture's Goal Drift failure path);
- autonomy-tier compliance rate — proportion of executed steps whose risk tier matched the human-approval requirement actually applied, confirming mixed-tier tasks did not silently execute a higher-tier step without its required checkpoint;
- delegation scope ratio — for tasks involving sub-agent delegation, granted sub-agent scope versus the delegating task's own scope, flagging cases approaching 1:1 (a signal of the Sub-Agent Scope Inheritance failure path).

### Retrieval-Augmented Generation

For workloads enforcing [EDR-0004](../engineering-decisions/EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md), per the [Enterprise RAG Architecture](../reference-architectures/ENTERPRISE_RAG_ARCHITECTURE.md):

- citation coverage rate — proportion of claims carrying a passage-identifier citation, versus flagged-ungrounded or refused;
- citation support rate — proportion of cited claims where the referenced passage actually supports the claim (distinct from citation coverage; a covered but unsupported citation is the "Citation Without Support" failure mode, and is tracked separately since it is worse than an uncited claim);
- retrieval precision/recall against a labeled query-passage reference set;
- passage freshness at time of use — proportion of cited passages within the workload's declared staleness target;
- cited-to-retrieved ratio — signal for context dilution from over-retrieval, per the RAG Architecture's corresponding failure path.

Citation coverage rate rising while citation support rate falls is a specific regression pattern worth naming: it means the system is citing more but citing worse, which the groundedness metric alone (an aggregate) would not surface as clearly as reading the two paired.

### Governance Compliance

Per [EDR-0005](../engineering-decisions/EDR-0005-USE-CASE-RISK-CLASSIFICATION-AND-APPROVAL-GATE.md):

- classification coverage rate — proportion of active production use cases with a current, recorded risk classification; this is a binary compliance metric, not a quality score, and its acceptable value is 100% — a use case either has a valid classification or it is a governance gap;
- accountable-owner currency rate — proportion of use cases with a confirmed, non-stale accountable owner, per the [AI Governance Review Playbook](../operational-playbooks/AI_GOVERNANCE_REVIEW_PLAYBOOK.md)'s ownership-handoff procedure;
- high-risk approval latency — time from a use case reaching High classification to recorded approval, monitored per EDR-0005's Approval Bottleneck failure mode so a growing queue is caught before teams route around the gate.

Unlike the quality and safety metrics above, these are not sampled — they are checked against the full population of active use cases, since a governance gap is binary per use case, not a rate to be estimated.

## Methodology

1. **Reference sets are versioned.** A reference/labeled set has an identifier and version; results are only comparable across runs against the same reference-set version.
2. **Every score has a denominator.** A pass rate is only reportable alongside the sample size and the population it was drawn from.
3. **Sampling is stratified by risk classification**, not uniform — high-risk workloads get proportionally more evaluation coverage than their traffic share.
4. **Human review uses calibrated rubrics**, not free-form judgment — reviewers score against a written rubric with worked examples at each score level, and inter-rater agreement is tracked.
5. **Evaluation results are attributed** to a specific model version, prompt version, and policy version — never to "the model" generically.

## Acceptance Criteria (Promotion Gates)

A model profile, prompt version, or policy change may be promoted to production routing only when:

- it has passed the offline benchmark at or above the workload class's minimum threshold;
- it has a nonzero, versioned reference set behind every reported score;
- no policy-compliance metric regressed against the currently active production version;
- latency at p95 is within the declared target for its model profile;
- a rollback path to the previous accepted version exists and has been verified (see the [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md)).

A promotion that meets every quantitative threshold but has no rollback path is not accepted — see Failure Modes below.

## Acceptance Thresholds by Model Profile

Thresholds are illustrative starting points; each organization must set and version its own, but every profile must have an explicit, non-empty threshold — "no threshold defined" is not a valid state for a production-eligible profile.

| Model Profile | Minimum Task Success | Minimum Groundedness | Maximum Policy Violation Rate |
|---|---|---|---|
| Economy | 90% | n/a unless retrieval-backed | 0% |
| Standard | 92% | 95% | 0% |
| Reasoning | 88% (harder task distribution) | 95% | 0% |
| Private | 92% | 95% | 0% |
| Multimodal | 85% (modality-dependent) | n/a unless retrieval-backed | 0% |

Policy violation rate has no acceptable nonzero threshold at any profile — this is a hard constraint, evaluated before optimization criteria, consistent with EDR-0002's Policy Evaluation rule.

## Failure Modes

### Evaluation Debt

A model profile ships to production before its offline benchmark exists, "to be evaluated later."

**Mitigation:** the routing layer treats "no evaluation record" as ineligible, not as a passing default.

### Reference Set Drift

The labeled reference set no longer represents production traffic, so passing scores no longer predict production quality.

**Mitigation:** reference sets are refreshed on a fixed schedule and whenever a production incident reveals an uncovered case; refreshes are versioned, not silent edits.

### Metric Gaming

A change improves the measured metric without improving the underlying task quality (e.g. shortening outputs to raise a formatting-compliance score at the cost of completeness).

**Mitigation:** no single metric gates promotion alone; task success and human review must agree with automated metrics before promotion.

### Silent Threshold Erosion

Thresholds are lowered under delivery pressure without a recorded decision.

**Mitigation:** threshold changes are versioned and require the same review as a policy change under EDR-0002 — a threshold is a decision, not a config value to edit informally.

## Related Decisions

- [EDR-0002 — Model Routing Strategy](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md) — evaluation results are one of the inputs to routing-policy eligibility.
