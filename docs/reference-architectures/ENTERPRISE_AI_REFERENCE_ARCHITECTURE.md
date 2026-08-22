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

---

### 4. Enterprise Services Layer

Connects AI with enterprise capabilities.

Examples:

- RAG
- Business Systems
- Knowledge Bases
- Internal APIs

---

### 5. Foundation Layer

Cross-cutting capabilities.

Includes:

- Governance
- Security
- Evaluation
- Observability

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