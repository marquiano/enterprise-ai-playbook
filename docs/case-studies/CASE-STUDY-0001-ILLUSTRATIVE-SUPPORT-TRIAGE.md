# Case Study 0001 — Illustrative: Support Ticket Triage

> **⚠️ Illustrative / Hypothetical.** This case study does not describe a real system, organization, or measured result. No implementation exists behind it. It exists solely to demonstrate the expected shape and rigor of a Case Study artifact under the [Engineering Artifact Catalog](../ENGINEERING_ARTIFACT_CATALOG.md), and to give the model-routing and evaluation artifacts in this Playbook a worked example to reference. It must not be cited as evidence, and it does not satisfy [ROADMAP.md](../roadmap.md)'s requirement for an end-to-end case study "linked to a runnable implementation" — that item remains open until a real implementation exists.

---

## Context (Hypothetical)

A hypothetical mid-size SaaS support organization receives a high volume of inbound tickets across billing, technical, and account-access categories. Triage is manual: a human router reads each ticket and assigns it to the correct queue and priority, at an assumed average handling cost per ticket that materially affects support-org headcount planning.

The hypothetical problem: manual triage is a bottleneck under this scenario, and its accuracy is inconsistent across shifts and reviewers.

## Hypothetical Implementation

The implementation, as hypothesized, would apply this Playbook's artifacts directly:

- **Model routing** — ticket triage is treated as a Standard-profile workload under [EDR-0002](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md): moderate reasoning need, low latency target, no elevated data sensitivity for most tickets, escalating to Reasoning profile only for ambiguous multi-issue tickets flagged by a low-confidence signal.
- **Architecture** — the triage service sits in the Enterprise Services Layer per the [Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md), calling Orchestration for classification and never calling a model provider directly, consistent with the Orchestration → Enterprise Services contract.
- **Evaluation** — classification accuracy, structured-output validity (queue and priority as a fixed schema), and policy-violation rate would be tracked per the [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md), gated by the Standard profile's illustrative acceptance thresholds before serving production traffic.
- **Operations** — routing and cost would be reviewed under the [Cost and Model-Routing Playbook](../operational-playbooks/COST_AND_MODEL_ROUTING_PLAYBOOK.md)'s weekly cadence, given triage's high request volume relative to other workloads.

## Hypothetical Results

**These figures are illustrative placeholders, not measurements.** A real case study must replace this entire section with sourced, dated, versioned data — sample size, reference-set version, and measurement window — per the Evaluation Strategy's Methodology section.

| Illustrative Metric | Hypothetical Before | Hypothetical After |
|---|---|---|
| Time to correct queue assignment | manual baseline (illustrative) | reduced, pending real measurement |
| Misrouted ticket rate | manual baseline (illustrative) | reduced, pending real measurement |
| Cost per ticket triaged | manual baseline (illustrative) | pending real measurement against Economy/Standard profile cost |

## Hypothetical Lessons Learned

Framed as what a real implementation would need to confirm or refute:

- Whether the Standard profile's illustrative 92% task-success threshold is actually appropriate for a queue-assignment task, or whether triage needs its own calibrated threshold — the Evaluation Strategy states thresholds are starting points, not fixed.
- Whether the escalation signal to the Reasoning profile (low-confidence flag) is well-calibrated, or routes too many/few tickets to the more expensive profile — this is exactly the kind of policy tuning the Cost and Model-Routing Playbook's Weekly Cost Review exists to catch.
- Whether a real incident under this workload would actually exercise the rollback procedure in the [Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md) as written, or reveal a gap in it.

## What Would Make This Real

This document graduates from illustrative to real only when it is rewritten against an actual running implementation, with:

- a real, named (or appropriately anonymized) organization and workload;
- dated, sourced measurements with a stated sample size and reference-set version;
- a link to the runnable implementation, per [EDR-0001 — Playbook Scope](../engineering-decisions/EDR-0001-PLAYBOOK-SCOPE.md), which requires implementations to live in dedicated repositories and be referenced from here, not embedded here.

## Related Artifacts

- [EDR-0002 — Model Routing Strategy](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md)
- [Enterprise AI Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md)
- [Enterprise AI Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)
- [Cost and Model-Routing Playbook](../operational-playbooks/COST_AND_MODEL_ROUTING_PLAYBOOK.md)
- [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md)
