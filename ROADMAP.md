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
| Agent tool-permission boundary decision | EDR-0003 | Proposed — held pending validation against a real implementation |
| Prompt injection / jailbreak defense pattern | Prompt Injection and Jailbreak Defense | Published |
| Agent architecture (planning, tool use, autonomy tiers) | Enterprise Agent Architecture | Published |
| Retrieval grounding and citation enforcement decision | EDR-0004 | Proposed — held pending validation against a real implementation |
| RAG architecture (ingestion, retrieval, grounded generation) | Enterprise RAG Architecture | Published |
| Use-case risk classification and approval gate decision | EDR-0005 | Proposed — held pending validation against a real implementation |
| Governance review process (intake, classification, approval, ownership) | AI Governance Review Playbook | Published |

## Next Evidence-Building Priorities

1. ~~Complete engineering decision record with alternatives and trade-offs.~~ Satisfied by EDR-0002 (formatting corrected); remains **Proposed** until validated against a real implementation.
2. ~~Worked reference architecture with contracts and failure paths.~~ Satisfied — see Enterprise AI Reference Architecture.
3. ~~Enterprise AI evaluation strategy with measurable acceptance criteria.~~ Satisfied — see Enterprise AI Evaluation Strategy.
4. ~~Cost and model-routing decision framework.~~ Satisfied — see EDR-0002 and the Cost and Model-Routing Playbook.
5. ~~Production incident and rollback playbook.~~ Satisfied — see AI Incident and Rollback Playbook.
6. **Still open:** end-to-end case study linked to a runnable implementation. Case Study 0001 is an illustrative/hypothetical placeholder only — it demonstrates the artifact shape but is explicitly not real evidence and does not satisfy this priority. This item remains open until a real, runnable implementation exists to document.

## Deferred Domains

The following areas remain intentionally unscaffolded until substantive content exists:

- observability;
- operations;
- case studies.

Security, evaluation, cost management, agent architecture, RAG, and governance are no longer deferred — see EDR-0003, the Prompt Injection and Jailbreak Defense pattern, the Enterprise AI Evaluation Strategy, the Cost and Model-Routing Playbook, the Enterprise Agent Architecture, EDR-0004, the Enterprise RAG Architecture, EDR-0005, and the AI Governance Review Playbook above. Security coverage to date is scoped to agent tool-permission boundaries and prompt injection/jailbreak; broader security topics (authentication/authorization architecture, data residency implementation, supply-chain risk) remain open. Agent architecture coverage is scoped to single- and delegated-agent planning/execution/autonomy; multi-agent coordination protocols beyond simple delegation remain open. RAG coverage is scoped to grounding/citation enforcement and the retrieval pipeline's structure; chunking-strategy trade-offs and multi-index federation remain open. Governance coverage is scoped to use-case risk classification and approval; broader governance topics (regulatory compliance mapping, board-level reporting, cross-organization policy harmonization) remain open.

## Delivery Rule

Every new artifact must answer:

1. What engineering problem does it solve?
2. Which decision, procedure or reusable model does it provide?
3. What evidence, constraints or trade-offs support it?
4. How can an engineering team apply it?
