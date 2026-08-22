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
