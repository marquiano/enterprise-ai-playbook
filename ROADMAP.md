# Enterprise AI Playbook Roadmap

## Purpose

This roadmap distinguishes implemented knowledge from planned work.

Planned topics are not considered part of the canonical Playbook until their supporting document is published and reviewed.

## Implemented

| Capability | Artifact | Status |
|---|---|---|
| Engineering principles | Enterprise AI Principles | Published |
| Delivery lifecycle | Enterprise AI Lifecycle | Published |
| System architecture | Enterprise AI Reference Architecture (with interface contracts and failure paths) | Published |
| Organizational assessment | Enterprise AI Maturity Model | Published |
| Knowledge organization | Enterprise AI Knowledge Graph | Published |
| Repository scope decision | EDR-0001 | Accepted |
| Model routing decision, with alternatives and trade-offs | EDR-0002 | Proposed — held pending validation against a real implementation, per its Review Notes |
| Evaluation strategy with measurable acceptance criteria | Enterprise AI Evaluation Strategy | Published |
| Cost and model-routing decision framework | EDR-0002 + Cost and Model-Routing Playbook | Published |
| Production incident and rollback playbook | AI Incident and Rollback Playbook | Published |

## Next Evidence-Building Priorities

1. ~~Complete engineering decision record with alternatives and trade-offs.~~ Satisfied by EDR-0002 (formatting corrected); remains **Proposed** until validated against a real implementation.
2. ~~Worked reference architecture with contracts and failure paths.~~ Satisfied — see Enterprise AI Reference Architecture.
3. ~~Enterprise AI evaluation strategy with measurable acceptance criteria.~~ Satisfied — see Enterprise AI Evaluation Strategy.
4. ~~Cost and model-routing decision framework.~~ Satisfied — see EDR-0002 and the Cost and Model-Routing Playbook.
5. ~~Production incident and rollback playbook.~~ Satisfied — see AI Incident and Rollback Playbook.
6. **Still open:** end-to-end case study linked to a runnable implementation. Case Study 0001 is an illustrative/hypothetical placeholder only — it demonstrates the artifact shape but is explicitly not real evidence and does not satisfy this priority. This item remains open until a real, runnable implementation exists to document.

## Deferred Domains

The following areas remain intentionally unscaffolded until substantive content exists:

- retrieval-augmented generation;
- agent architecture;
- security;
- governance;
- evaluation;
- observability;
- operations;
- cost management;
- case studies.

## Delivery Rule

Every new artifact must answer:

1. What engineering problem does it solve?
2. Which decision, procedure or reusable model does it provide?
3. What evidence, constraints or trade-offs support it?
4. How can an engineering team apply it?
