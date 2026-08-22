# Enterprise RAG Architecture

> Reference Architecture — the internal structure of a retrieval-augmented generation system operating inside the Enterprise Services Layer of the [Enterprise AI Reference Architecture](ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md).

---

## Relationship to the Enterprise AI Reference Architecture

The main Reference Architecture lists RAG as an example capability of the Enterprise Services Layer, alongside business systems, knowledge bases, and internal APIs. This document defines what that capability looks like internally when a workload requires it, and it enforces [EDR-0004](../engineering-decisions/EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md)'s grounding decision structurally rather than restating it.

## Components

### Ingestion and Indexing Pipeline

Converts source documents into retrievable form: chunking, embedding, and indexing. This is where staleness originates — a source that changes after ingestion is not reflected in the index until the pipeline re-processes it.

### Retriever

Given a request, returns candidate passages from the index ranked by relevance. The retriever's output is untrusted content by the definition already established in the [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md) pattern — whatever the retriever surfaces is processed content, not an instruction, regardless of what it contains.

### Re-ranker

An optional second-stage component that re-scores the retriever's candidates against the specific request before passages are passed to generation. Where present, it is the last point at which an irrelevant or low-quality passage can be excluded before it reaches the generator.

### Grounded Generator

Produces the response, constrained per [EDR-0004](../engineering-decisions/EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md) to reference specific passage identifiers for each claim, and to refuse or flag claims it cannot ground.

## Contracts

### Ingestion/Indexing Pipeline → Retriever

**Provides:** an index of passages, each carrying a source identifier and an ingestion/last-verified timestamp.

**Guarantees:** the retriever can report, for any passage it surfaces, how current that passage is — freshness is not discarded at indexing time.

### Retriever → Re-ranker / Generator

**Provides:** ranked candidate passages with their source identifiers and freshness metadata, tagged as retrieved content per the injection-defense pattern's provenance-tagging control.

**Guarantees:** passages are passed as structured, identified units — not concatenated into undifferentiated free text the generator reads as part of its own reasoning trace, consistent with the injection-defense pattern's data/instruction separation control.

### Re-ranker / Retriever → Grounded Generator

**Provides:** the final passage set the generator is permitted to cite.

**Guarantees:** the generator cannot cite a passage that was not in this set — citation is a reference into a known, bounded list, not free-text the generator could fabricate.

### Grounded Generator → Response Consumer

**Provides:** a response where each claim is either linked to a specific passage identifier, explicitly flagged as ungrounded, or the response is a refusal stating no relevant context was found.

**Guarantees:** per EDR-0004's Acceptance Criteria, the consumer can distinguish "grounded," "flagged ungrounded," and "refused — no context" as three distinct, never-conflated response states.

## Failure Paths

### Index Staleness

**Cause:** a source document changes or is retracted after ingestion, but the index still surfaces the outdated version, and the generator grounds a claim in it.

**Detection:** compare a passage's last-verified timestamp against its source's actual last-modified time at query time, where the source system exposes one; monitor re-ingestion lag against a declared freshness target per workload.

**Mitigation:** declare a maximum acceptable staleness per workload (analogous to a latency target in EDR-0002); passages older than that target are either re-verified before use or excluded from the candidate set, not silently served as current.

### Silent Ungrounded Generation

**Cause:** the generator produces a claim with no supporting passage, and no enforcement mechanism catches it — the failure mode [EDR-0004](../engineering-decisions/EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md) exists specifically to prevent.

**Detection:** groundedness evaluation (per the [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)) sampling claims against cited passages; a claim with no citation in a workload where EDR-0004 enforcement is supposedly active is itself a signal the enforcement mechanism is not correctly wired in.

**Mitigation:** grounding constraints are enforced in the generation step's structure (passage-identifier referencing), not left to a prompt instruction asking the model to cite sources — the same reasoning EDR-0004 already gives for rejecting Alternative B.

### Context Poisoning via Untrusted Retrieved Content

**Cause:** a retrieved document contains content crafted to manipulate the generator — e.g., instructions embedded in a document designed to make the generator assert a specific false claim "grounded" in that same document.

**Detection:** this is the retrieval-specific instance of the failure path already documented in the main Reference Architecture ("Prompt Injection Drives Unauthorized Tool Invocation") and the injection-defense pattern's adversarial reference set; detection follows the same mechanism.

**Mitigation:** the injection-defense pattern's controls apply to retrieved content regardless of whether grounding is enforced — grounding enforcement verifies a claim traces to a passage, it does not verify the passage itself is trustworthy. Both controls are required together, as EDR-0004's corresponding Failure Mode states.

### Context Dilution from Over-Retrieval

**Cause:** the retriever returns more passages than the generator can meaningfully use, diluting relevant content among irrelevant candidates and increasing the chance a claim gets grounded in a marginally-relevant passage rather than the best available one.

**Detection:** monitor the ratio of cited-to-retrieved passages; a consistently low ratio suggests over-retrieval rather than generator failure.

**Mitigation:** the re-ranker component exists specifically to narrow the candidate set before generation; a workload retrieving large candidate sets without a re-ranking stage is a design gap, not an acceptable default.

## Related Artifacts

- [Enterprise AI Reference Architecture](ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md)
- [EDR-0004 — Retrieval Grounding and Citation Enforcement](../engineering-decisions/EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md)
- [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md)
- [Enterprise AI Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)
- [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md)
