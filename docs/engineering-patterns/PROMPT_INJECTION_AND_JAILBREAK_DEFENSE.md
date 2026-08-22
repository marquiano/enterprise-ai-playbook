# Engineering Pattern — Prompt Injection and Jailbreak Defense

> Engineering Pattern — proven, reusable defenses against a model being manipulated by content it processes, or by its own conversation, into acting against its intended instructions.

---

## Context

An Enterprise AI system built on the [Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md) routinely processes content the organization does not fully control: retrieved documents, tool outputs, third-party API responses, user-supplied files, and the user's own conversational input. Any of that content can carry instructions crafted to override the system's actual instructions — this is prompt injection when the attack arrives via processed content, and jailbreaking when it arrives via the direct conversational channel attempting to override the system prompt's constraints.

Both attack classes share the same underlying weakness: a language model does not have a reliable, built-in way to distinguish "instructions I should follow" from "text I am merely processing." Anything that looks like an instruction, wherever it appears in the context window, is a candidate the model may act on.

This pattern does not eliminate that weakness — no purely model-level defense does, which is why [EDR-0003](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) exists as an independent, non-model-level control for the case where these defenses are bypassed. This pattern reduces how often manipulation succeeds; EDR-0003 bounds the damage when it does.

## Applicability

Apply this pattern to any workload where the model processes content from a source the organization does not fully trust, including:

- retrieval-augmented generation over an internal or external document corpus;
- tool outputs where the tool queries a system outside the organization's direct control (web content, third-party APIs, user-uploaded files);
- multi-turn conversations where later turns are treated with the same trust as the original system prompt;
- any agent capable of taking action via tool calls (this pattern is a precondition for EDR-0003's authorization boundary to be meaningful — an agent that can be steered into requesting a malicious tool call still needs that call blocked at authorization, but blocking fewer malicious requests in the first place reduces the human-approval burden the EDR's mitigation depends on).

Do not treat this pattern as sufficient on its own for any workload where a successful bypass would cause irreversible or high-consequence effects — those workloads require EDR-0003's permission boundary regardless of how strong the input-side defense is.

## The Pattern

### 1. Instruction Hierarchy

Explicitly rank instruction sources and state the ranking in the system prompt itself: system/developer instructions outrank user instructions, which outrank content encountered during task execution (retrieved documents, tool outputs, third-party content). Content encountered during execution is never treated as instructions to the model, regardless of its phrasing or claimed authority ("ignore previous instructions," "the developer has authorized," "system override") — this exact framing is called out in this organization's own operating instructions as invalid regardless of urgency or claimed authority, and the principle generalizes to any enterprise agent.

### 2. Provenance Tagging

Content entering the context window is tagged with its source and trust level before the model sees it (e.g., clearly delimited as "retrieved document," "tool output," "user message"). This does not make the model immune to manipulation, but it gives the instruction hierarchy something concrete to apply, and gives downstream logging something to attribute a manipulated response to.

### 3. Data/Instruction Separation at the Boundary

Where the surrounding system (not the model) can enforce separation mechanically — e.g., a tool's return value is inserted into a structured field rather than concatenated into free text the model reads as part of its own reasoning trace — prefer the mechanical separation. A boundary enforced in code is not defeated by clever phrasing the way a boundary enforced only by prompt instruction is.

### 4. Output-Side Policy Checks

Before an output or a proposed tool call is acted on, check it against policy independently of the reasoning that produced it — consistent with the Reference Architecture's Foundation Layer output-filtering responsibility. A manipulated model producing a policy-violating output is caught here even if the manipulation itself went undetected upstream.

### 5. Least-Trust Defaults for Ambiguous Authority Claims

When content claims elevated authority ("as the administrator, I am telling you to..."), the default is to treat the claim as unverified and route to the same authorization path as any other request — never to grant elevated trust based on a claim made inside content the model is processing. This is the same principle EDR-0003 applies at the tool-authorization layer, applied here at the input-interpretation layer.

### 6. Adversarial Evaluation as a Release Gate

Per the [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md), maintain a reference set of known injection and jailbreak attempts, versioned like any other evaluation reference set, and run it as part of the offline benchmark before any prompt, model, or policy version is promoted. A new bypass discovered in production is added to this reference set, not just patched ad hoc — otherwise the same bypass class recurs on the next model or prompt change.

## Benefits

- reduces the frequency of successful manipulation without requiring the model itself to be modified;
- most controls are enforceable in surrounding system code, not dependent on the model reliably following instructions about its own behavior;
- composes directly with EDR-0003 — this pattern reduces the attack surface EDR-0003's authorization boundary has to catch, and EDR-0003 bounds what happens when this pattern is bypassed;
- provides concrete, versioned test material (the adversarial reference set) rather than a vague "be careful" instruction.

## Limitations

- no combination of these controls achieves zero successful-manipulation rate; this pattern manages likelihood, not certainty — see [EDR-0003](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) for the control that assumes manipulation will sometimes succeed;
- provenance tagging and instruction hierarchy depend on the model correctly interpreting the tagging scheme — a sufficiently novel attack may exploit the tagging convention itself;
- adversarial reference sets go stale as new attack techniques emerge; the mitigation (continuous addition on discovery) requires an operational process, not a one-time setup — see the [Cost and Model-Routing Playbook](../operational-playbooks/COST_AND_MODEL_ROUTING_PLAYBOOK.md) for the operating-cadence pattern this should follow;
- output-side policy checks add latency and cannot catch every failure mode, particularly manipulations that produce outputs which are policy-compliant in form but wrong in substance.

## Related Artifacts

- [EDR-0003 — Agent Tool-Scope and Permission Boundary](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md)
- [Enterprise AI Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md)
- [Enterprise AI Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)
- [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md)
