# EDR-0004 — Retrieval Grounding and Citation Enforcement

## Status

Proposed

## Context

Retrieval-augmented generation (RAG) exists in the [Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md)'s Enterprise Services Layer to supply a model with organization-specific context it was not trained on. The premise is that a response grounded in retrieved context is more trustworthy than one produced from the model's parametric knowledge alone.

That premise only holds if grounding is actually enforced. A model given retrieved context is not thereby prevented from producing claims the retrieved context does not support — it can still generate plausible-sounding statements from its own parametric knowledge, blend them with retrieved facts indistinguishably, or misattribute a claim to a source that does not actually support it. The [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md) already tracks "groundedness" as a metric for retrieval-backed workloads, but a metric that is measured after the fact does not itself enforce anything at generation time — it tells you how often the problem occurred, not that it did not occur for this specific response.

The organization must decide whether grounding and citation are enforced as a generation-time constraint, or treated only as a quality signal to be measured and improved over time.

## Decision Drivers

The grounding approach must:

- make it possible to verify, for any specific claim in a response, whether it is supported by retrieved context;
- prevent (not merely detect) a response from asserting a claim with no supporting retrieved context, for workloads where that matters;
- allow legitimate use of the model's general knowledge where the workload does not require grounding, without forcing every response into a citation format that adds no value;
- remain auditable — which source, of potentially many retrieved chunks, supports which claim;
- degrade safely — when no retrieved context supports the request, the system says so rather than filling the gap with an ungrounded answer;
- avoid citation as theater — a citation attached to a claim it does not actually support is worse than no citation, because it manufactures false confidence.

## Decision

Adopt claim-level grounding enforcement for any workload classified as retrieval-backed (per the [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)'s retrieval-backed groundedness threshold): each factual claim in a generated response must be traceable to a specific retrieved source, and the generation step must refuse or explicitly flag any claim it cannot ground, rather than asserting it.

This is enforced at generation time through structural constraints, not left to a post-hoc quality check:

- the generator is constrained to produce output referencing specific retrieved passages (by identifier), not free-text citation the model could fabricate;
- a claim with no corresponding referenced passage is treated as ungrounded and either suppressed, flagged to the reader, or triggers a refusal for the whole response, depending on workload risk tier;
- "no relevant context was retrieved" is a valid, first-class response — the generator does not default to answering from parametric knowledge when retrieval returns nothing useful, unless the workload has explicitly declared that fallback as acceptable (see Reversal-adjacent case in Acceptance Criteria).

Workloads not classified as retrieval-backed are unaffected by this decision — general-knowledge workloads are not forced into a citation structure that does not apply to them.

## Alternatives Considered

### Alternative A — Post-Hoc Groundedness Measurement Only

Measure groundedness as an evaluation metric (already the case per the Evaluation Strategy) without any generation-time enforcement; treat a low groundedness score as a signal to improve prompts or retrieval quality over time.

**Advantages**

- no generation-time latency or complexity cost;
- simplest to implement, since it requires no change to the generation step itself.

**Rejected Because**

- a metric measured in aggregate, on a sample, does not prevent any individual response from asserting an ungrounded claim — the failure still reaches the user before it is caught;
- provides no mechanism for the reader to verify a specific claim in the response they received, only an aggregate quality trend.

### Alternative B — Optional, Model-Generated Citations

Ask the model to include citations in its free-text output when it deems appropriate, without structural constraint or verification.

**Advantages**

- easy to prompt for;
- no change to retrieval or generation infrastructure.

**Rejected Because**

- a model-generated citation is not verified against what was actually retrieved — the model can cite a source that does not say what the claim asserts, which is the "citation as theater" failure named in the Decision Drivers;
- citation becomes inconsistent across responses since it depends on the model's judgment of when to include one, not a fixed policy.

### Alternative C — Mandatory Human Review of All RAG Outputs

Require a human reviewer to check every RAG-generated response against its retrieved sources before it reaches the end user.

**Advantages**

- highest achievable accuracy of grounding verification;
- catches errors the structural approach might miss.

**Rejected Because**

- does not scale to the request volume most RAG workloads are deployed for — the review bottleneck defeats the purpose of automating the retrieval-and-answer workflow;
- creates the same approval-fatigue risk already documented in [EDR-0003](EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) for high-volume human checkpoints.

## Consequences

**Positive**

- a reader can verify any specific claim against its cited source, rather than trusting the response as a whole;
- ungrounded claims are caught before reaching the user for enforcement-covered workloads, not only measured afterward;
- "no relevant context" becomes an explicit, honest response state instead of a silent fallback to parametric knowledge.

**Negative**

- claim-level grounding constrains generation and adds latency compared to unconstrained free-text generation;
- retrieval quality becomes a harder dependency — a workload with poor retrieval coverage will produce more refusals or flagged claims, which is correct behavior but may be perceived as reduced usefulness;
- structural citation requires generation and retrieval infrastructure that supports passage-level referencing, which is more implementation effort than free-text citation.

## Failure Modes

### Citation Without Support

A generated claim references a retrieved passage that does not actually support it (misattribution rather than fabrication of the citation mechanism itself).

Mitigation:

- groundedness evaluation (per the Evaluation Strategy) samples cited claims against their referenced passages specifically, not just checking that a citation exists;
- this failure is tracked as a distinct evaluation category from "no citation provided," since it is arguably worse — it manufactures false confidence.

### Retrieval Starves Legitimate Answers

Retrieval returns no useful context for a question the organization's knowledge base could answer if queried differently, and the generator correctly refuses — but the refusal is indistinguishable from a case where no answer exists at all.

Mitigation:

- refusal responses distinguish "no supporting context was found" from "the retrieved context contradicts the claim," so the operational team can tell whether the gap is a retrieval problem or a genuine knowledge gap;
- retrieval quality (precision/recall, per the Evaluation Strategy's RAG metrics) is monitored separately from generation-time grounding compliance, so a rising refusal rate is investigated as a retrieval issue rather than treated as expected behavior.

### Stale Retrieved Context

An indexed source has changed or been retracted since ingestion, but the retriever still surfaces the outdated version, and the generator grounds a claim in now-incorrect content — see the corresponding failure path in the [Enterprise RAG Architecture](../reference-architectures/ENTERPRISE_RAG_ARCHITECTURE.md).

Mitigation:

- index freshness is tracked and surfaced per the Enterprise RAG Architecture's Ingestion/Indexing component contract; grounding enforcement verifies a claim traces to a passage, but does not by itself verify that passage is current.

### Context Poisoning via Untrusted Retrieved Content

A retrieved document itself contains adversarial content designed to manipulate the generator — this is the retrieval-specific case of the [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md) pattern's threat model, and grounding enforcement alone does not defend against it: a generator can faithfully "ground" a claim in a passage that was itself crafted to produce that exact false claim.

Mitigation:

- grounding enforcement and the injection-defense pattern are complementary, not substitutes for each other — retrieved content is still subject to the pattern's provenance tagging and instruction-hierarchy controls regardless of whether grounding is enforced.

## Observability Requirements

Each grounding-enforced generation must record:

- which retrieved passages were available to the generator;
- which passages were actually cited per claim;
- any claim suppressed or flagged as ungrounded, and why;
- whether the response was a full refusal due to no relevant retrieved context.

## Acceptance Criteria

The decision is considered successfully implemented when:

- every claim in a retrieval-backed workload's response is traceable to a specific retrieved passage identifier, or is explicitly flagged/suppressed as ungrounded;
- "no relevant context retrieved" produces an explicit refusal response, not a silent fallback to parametric knowledge, unless a workload has explicitly and separately declared parametric fallback as an acceptable, documented behavior for that specific workload;
- groundedness evaluation samples verify cited claims against their referenced passages, not merely the presence of a citation;
- refusal responses distinguish "no context found" from "context contradicts the claim."

## Reversal Criteria

This decision should be reconsidered if:

- claim-level enforcement is shown, with real usage data, to degrade response usefulness (e.g., excessive refusal rate) more than it improves trustworthiness for the workloads it covers;
- retrieval and generation infrastructure cannot support passage-level referencing at acceptable latency for a given workload, and no equivalent enforcement mechanism is available;
- evaluation data shows unenforced (Alternative A) groundedness rates are already acceptable for a given workload's risk tier, making enforcement unnecessary overhead for that specific case.

## Related Decisions

- [EDR-0002 — Model Routing Strategy](EDR-0002-MODEL-ROUTING-STRATEGY.md) — groundedness is one of the acceptance thresholds a model profile must meet for retrieval-backed workloads.
- [EDR-0003 — Agent Tool-Scope and Permission Boundary](EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) — the precedent for enforcing a constraint structurally rather than relying on the model's own compliance.
- [Enterprise RAG Architecture](../reference-architectures/ENTERPRISE_RAG_ARCHITECTURE.md) — the components and contracts this decision is enforced within.
- [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md) — the complementary control for adversarial retrieved content.

## Review Notes

This decision must be validated against at least one runnable implementation before being marked Accepted, consistent with the review discipline already applied to EDR-0002 and EDR-0003.
