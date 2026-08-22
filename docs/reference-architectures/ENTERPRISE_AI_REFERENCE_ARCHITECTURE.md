# Enterprise AI Reference Architecture

> Canonical reference architecture for Enterprise AI Systems.

---

## Layers

### 1. User Layer

Consumers of AI capabilities.

Examples:

- Employees
- Customers
- Partners
- Applications

---

### 2. Interaction Layer

Responsible for exposing AI capabilities.

Examples:

- APIs
- Chat Interfaces
- Web Applications
- Mobile Applications

---

### 3. AI Orchestration Layer

Coordinates AI execution.

Examples:

- Agent Frameworks
- Prompt Management
- Workflow Engines
- Tool Routing

When this layer is agentic — planning across multiple steps rather than producing a single response — see the [Enterprise Agent Architecture](ENTERPRISE_AGENT_ARCHITECTURE.md) for its internal structure, contracts, and failure paths.

---

### 4. Enterprise Services Layer

Connects AI with enterprise capabilities.

Examples:

- RAG
- Business Systems
- Knowledge Bases
- Internal APIs

Where this layer provides retrieval-augmented generation, see the [Enterprise RAG Architecture](ENTERPRISE_RAG_ARCHITECTURE.md) for its internal structure, contracts, and failure paths.

---

### 5. Foundation Layer

Cross-cutting capabilities.

Includes:

- Governance
- Security
- Evaluation
- Observability

Governance's use-case classification and approval gate is decided in [EDR-0005](../engineering-decisions/EDR-0005-USE-CASE-RISK-CLASSIFICATION-AND-APPROVAL-GATE.md) and operationalized by the [AI Governance Review Playbook](../operational-playbooks/AI_GOVERNANCE_REVIEW_PLAYBOOK.md). Observability's cross-component tracing and alerting structure is defined in the [Enterprise AI Observability Pattern](../engineering-patterns/ENTERPRISE_AI_OBSERVABILITY_PATTERN.md).

---

### 6. Infrastructure Layer

Execution platform.

Examples:

- Cloud
- Kubernetes
- Models
- Vector Databases
- Storage

---

## Interface Contracts

Each layer boundary is a contract, not just an adjacency. A layer must not depend on the internal implementation of the layer below it — only on the contract it publishes.

### User Layer → Interaction Layer

**Provides:** authenticated user or system identity, request intent, input payload.

**Guarantees:** the Interaction Layer authenticates and authorizes the caller before any request reaches AI Orchestration; unauthenticated or unauthorized requests never cross this boundary.

### Interaction Layer → AI Orchestration Layer

**Provides:** a normalized request (see [EDR-0002](../engineering-decisions/EDR-0002-MODEL-ROUTING-STRATEGY.md) for an example of a normalized inference request shape: task type, risk classification, latency target, quality threshold, data sensitivity).

**Guarantees:** Orchestration never receives raw, unvalidated user input directly from a channel adapter — the Interaction Layer owns input sanitization and channel-specific concerns (rate limiting, session state, protocol translation).

### AI Orchestration Layer → Enterprise Services Layer

**Provides:** a tool or data invocation with declared scope (what data, what system, what operation) rather than an open-ended query.

**Guarantees:** Enterprise Services enforce their own authorization independently of what Orchestration claims — Orchestration is a caller, not a trust boundary.

### Enterprise Services Layer → Foundation Layer

**Provides:** every enterprise-service call and its result flow through the Foundation Layer's observability and evaluation hooks.

**Guarantees:** Governance, Security, Evaluation and Observability are cross-cutting — no layer may bypass them by calling Infrastructure directly.

### Foundation Layer → Infrastructure Layer

**Provides:** policy decisions (allow/deny, routing selection, redaction rules) applied before Infrastructure is invoked.

**Guarantees:** Infrastructure never receives a request that has not already passed governance and security policy evaluation.

### Any Layer → Infrastructure Layer

**Provides:** N/A — this path must not exist. All layers above Foundation reach Infrastructure only through the Foundation Layer's policy enforcement point.

**Guarantees:** a request cannot reach a model, vector database, or storage system by skipping Governance and Security.

---

## Failure Paths

Each boundary above has a corresponding way it can fail. A reference architecture that only describes the success path is incomplete.

### Unauthenticated Request Reaches Orchestration

**Cause:** Interaction Layer misconfiguration, or a direct call bypassing the Interaction Layer.

**Detection:** Orchestration rejects requests without a valid identity claim; missing-identity rejections are logged and alertable.

**Mitigation:** identity verification is enforced at the Orchestration boundary as well as at Interaction — defense in depth, not a single checkpoint.

### Orchestration Calls a Model Provider Directly

**Cause:** an application or agent bypasses the routing layer defined in EDR-0002, calling a provider SDK directly.

**Detection:** network egress monitoring for direct provider endpoints from services that should only reach the routing layer.

**Mitigation:** provider credentials are not distributed to Orchestration-layer services; only the routing layer holds them.

### Enterprise Service Trusts Orchestration's Claims

**Cause:** an Enterprise Service skips its own authorization check because "Orchestration already validated it."

**Detection:** authorization audit shows enterprise-service calls with no independent policy evaluation recorded.

**Mitigation:** every Enterprise Service call is independently authorized; Orchestration-layer identity is necessary but never sufficient.

### Foundation Layer Bypassed Under Load

**Cause:** during incidents or load spikes, a shortcut path is added "temporarily" from Enterprise Services straight to Infrastructure to reduce latency, skipping observability and policy checks.

**Detection:** infrastructure access logs show request volume with no corresponding Foundation Layer decision record.

**Mitigation:** no code path may reach Infrastructure without a Foundation Layer decision, including under degraded conditions — degraded mode must fail closed, not bypass policy. See the [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md) for the corresponding operational procedure.

### Silent Cross-Layer Data Leakage

**Cause:** a response assembled in Enterprise Services carries sensitive fields that Interaction or User layers were never authorized to see, because no layer enforces output-side filtering.

**Detection:** evaluation and observability tooling (Foundation Layer) samples responses against data-sensitivity policy.

**Mitigation:** output filtering is a Foundation Layer responsibility applied on the way back up, not only on the way down.

### Prompt Injection Drives Unauthorized Tool Invocation

**Cause:** content processed by AI Orchestration (a retrieved document, a tool result, third-party content) carries instructions that manipulate the model into proposing a tool call outside the intent of the original request. See the [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md) pattern for input-side mitigation.

**Detection:** the Enterprise Services Layer's independent authorization check (already required by the Orchestration → Enterprise Services contract above) rejects a proposed call whose declared scope exceeds what the requesting task's per-task grant allows, per [EDR-0003](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md).

**Mitigation:** the model's proposed tool call is never treated as self-authorizing — EDR-0003's per-task permission boundary bounds the damage even when the injection defense above is bypassed. This is why the Orchestration → Enterprise Services contract requires independent authorization rather than trusting Orchestration's claim.

### Agent Tool-Scope Overreach and Data Exfiltration

**Cause:** an agent is granted (or defaults to) a tool scope broader than its current task requires, and either a manipulated model or a reasoning error uses that excess scope to read data unrelated to the task or to send data to an unintended destination.

**Detection:** audit comparison of granted scope versus actually-used scope (per EDR-0003's Observability Requirements) surfaces systematic over-grants; anomalous outbound data volume or destination on a tool call is a runtime signal.

**Mitigation:** EDR-0003's scoped, per-task, expiring grants with human approval for irreversible-or-external-effect operations — a static, broad grant is the precondition this failure mode requires, and EDR-0003 exists specifically to remove that precondition.

### Runaway Agent Execution or Goal Drift

**Cause:** an agentic AI Orchestration Layer's planning loop drifts from the original task intent over many steps, or gets stuck proposing steps without making progress — see the [Enterprise Agent Architecture](ENTERPRISE_AGENT_ARCHITECTURE.md) Failure Paths for the detailed mechanism.

**Detection:** step count and resource usage tracked against the task's declared budget; proposed-step intent compared against original task intent at fixed intervals.

**Mitigation:** hard step/resource budgets terminate the task rather than allowing indefinite continuation, and a flagged intent divergence is treated as requiring reauthorization, not silently allowed to proceed — both per the Enterprise Agent Architecture's contracts.

### Stale or Poisoned Retrieval Context

**Cause:** a retrieval-augmented workload in the Enterprise Services Layer grounds a response in retrieved content that is either outdated or adversarially crafted — see the [Enterprise RAG Architecture](ENTERPRISE_RAG_ARCHITECTURE.md)'s "Index Staleness" and "Context Poisoning via Untrusted Retrieved Content" failure paths for the detailed mechanism.

**Detection:** passage freshness monitoring against a declared staleness target; groundedness evaluation sampling (per the [Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)) flagging claims grounded in content that does not actually support them.

**Mitigation:** [EDR-0004](../engineering-decisions/EDR-0004-RETRIEVAL-GROUNDING-AND-CITATION.md)'s structural grounding enforcement plus the [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md) pattern's provenance controls — grounding enforcement alone does not defend against poisoned content, and injection defense alone does not defend against stale content, so both apply together.

### Unclassified or Unapproved Use-Case Reaches Production

**Cause:** a use case obtains model routing (EDR-0002) or tool authorization (EDR-0003) without first completing the risk classification and, if high-risk, approval required by [EDR-0005](../engineering-decisions/EDR-0005-USE-CASE-RISK-CLASSIFICATION-AND-APPROVAL-GATE.md) — typically because it reached those layers through a path that does not check for a registered governance record.

**Detection:** the [AI Governance Review Playbook](../operational-playbooks/AI_GOVERNANCE_REVIEW_PLAYBOOK.md)'s periodic reclassification review cross-references active routing/authorization traffic against registered use cases; any traffic from an unregistered use-case identity is a gap by definition.

**Mitigation:** EDR-0002's routing layer and EDR-0003's tool-authorization layer both check for a registered, classified use-case identity before granting access — the enforcement point is structural (no registration, no access), not merely procedural (a review that happens to catch it eventually).

### Observability Blind Spot Prevents Root-Causing

**Cause:** a component in the request path does not propagate the shared trace identifier defined in the [Enterprise AI Observability Pattern](../engineering-patterns/ENTERPRISE_AI_OBSERVABILITY_PATTERN.md), so its records cannot be correlated with the rest of the request — an incident responder can see the routing decision, the tool authorization, or the retrieval/citation record, but not how they relate to each other for that specific request.

**Detection:** trace-level reconstruction (per the Observability Pattern's Control 3) fails to produce a complete cross-component picture for a sampled set of traces; a component's records exist but never appear correlated to any trace identifier.

**Mitigation:** trace propagation is verified as part of any new component's integration, not assumed — the Observability Pattern's Limitations section already names this as its core dependency: one component dropping the identifier breaks reconstruction for every request passing through it.