# EDR-0003 — Agent Tool-Scope and Permission Boundary

## Status

Proposed

## Context

The [Reference Architecture](../reference-architectures/ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md) places tool and data invocation in the AI Orchestration Layer's call into the Enterprise Services Layer. That contract states the invocation must carry a "declared scope (what data, what system, what operation)" and that Enterprise Services must authorize independently of Orchestration's claims — but it does not decide *how* an agent acquires the permission to invoke a tool in the first place, or how broad that permission is allowed to be.

In practice, agent frameworks tend toward one of two extremes:

- **static, broad grants** — an agent is configured once with a fixed set of tools and effectively unrestricted parameters, because narrowing scope per-call is engineering effort nobody prioritizes;
- **no formal grant at all** — any tool registered with the orchestration runtime is callable by any agent session, because the runtime does not distinguish "available" from "authorized for this task."

Both extremes create the same class of failure: a model, manipulated through crafted input (prompt injection) or simply through its own reasoning error, invokes a tool with a scope broader than the current task requires — reading a data source unrelated to the request, or invoking a write/send/delete operation nothing in the task justified.

This is distinct from — and a precondition for — the [Prompt Injection and Jailbreak Defense pattern](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md): input-side defenses reduce how often a model is manipulated, but a permission boundary limits the *damage* even when manipulation succeeds. Both are necessary; neither substitutes for the other.

## Decision Drivers

The permission boundary must:

- limit tool-call authorization to what the current task actually requires, not what the agent's configuration generically allows;
- keep the authorization decision inside the trust boundary (Enterprise Services or a dedicated policy point), never inferred from what Orchestration or the model claims;
- require explicit human approval before any high-risk action executes, where "high-risk" is defined by consequence (irreversible, financial, external-facing, or affecting another party's data) rather than by tool name alone;
- produce an auditable record of every grant, every denial, and every human approval;
- fail closed — an ungranted or ambiguous permission is a denial, not a default allow;
- remain enforceable independently of prompt content, since prompt content is the one input an attacker fully controls.

## Decision

Adopt scoped, per-task tool permission grants, issued by the Enterprise Services Layer (or a dedicated policy enforcement point it delegates to), not configured statically per agent.

A tool-call authorization must specify:

- the requesting task's identity and declared intent;
- the specific tool and operation (not "the toolset");
- the specific data or system scope the operation may touch;
- a risk classification (read-only / reversible-write / irreversible-or-external-effect);
- an expiry — a grant is valid for the current task execution, not indefinitely.

Irreversible-or-external-effect operations (sending a message, executing a financial transaction, deleting data, modifying a production system) require a human-in-the-loop approval step before execution, regardless of how confident the model's output appears. This mirrors the "explicit permission required" category already applied to Claude's own action-taking behavior elsewhere in this organization's tooling — the same principle applies to any enterprise agent, not just this one.

The model's own output requesting a tool call is treated as a *proposal*, never as an authorization. Authorization is decided against the policy above, independently of what the model claims the call is for.

## Alternatives Considered

### Alternative A — Static Per-Agent Tool Grants

Each agent is configured once with a fixed list of tools it may call, checked at registration time rather than per call.

**Advantages**

- simple to implement;
- low runtime overhead;
- easy to reason about at design time.

**Rejected Because**

- grants do not narrow to the current task — an agent authorized to read a data source for one workflow can read it for any workflow it runs, including one a prompt injection steered it into;
- no expiry — a grant issued for one session persists indefinitely;
- cannot express "this specific record" scoping, only "this tool."

### Alternative B — Model Self-Restraint via System Prompt

Rely on instructions in the system prompt telling the model which tools to use and when, with no independent enforcement.

**Advantages**

- zero additional infrastructure;
- fast to iterate on.

**Rejected Because**

- a system prompt is not an enforcement boundary — it is content the model reasons over, and reasoning over content is exactly what prompt injection exploits;
- provides no audit trail of what was actually authorized versus what was instructed;
- violates the Decision Driver that enforcement must be independent of prompt content.

### Alternative C — Full Human Approval for Every Tool Call

Require human sign-off before any tool call executes, regardless of risk classification.

**Advantages**

- maximal safety;
- simplest policy to state.

**Rejected Because**

- makes autonomous agent workflows non-viable in practice — the approval bottleneck defeats the purpose of automation for the large majority of low-risk, reversible calls;
- human reviewers facing high call volume habituate to approving quickly, degrading the control's actual effectiveness (approval fatigue) — a documented failure mode of blanket-approval designs.

## Consequences

**Positive**

- damage from a successful prompt injection or reasoning error is bounded by the narrowest applicable scope, not by the agent's full tool inventory;
- every tool invocation is attributable to a specific authorized task and, for high-risk actions, a specific human approver;
- policy can tighten or loosen scope centrally without redeploying agent configuration, consistent with the routing-policy pattern established in [EDR-0002](EDR-0002-MODEL-ROUTING-STRATEGY.md).

**Negative**

- per-task authorization adds latency and infrastructure the static-grant alternative avoids;
- risk classification requires ongoing maintenance as new tools and operations are added — a misclassified irreversible action as "reversible" defeats the control;
- human-in-the-loop steps introduce an operational dependency (approver availability) that must itself have a defined escalation path.

## Failure Modes

### Misclassified Risk Tier

An operation capable of irreversible or external effect is classified as reversible, so it executes without human approval.

Mitigation:

- risk classification is reviewed whenever a tool's capability changes, not only at initial registration;
- default classification for any new or unclassified tool is the highest tier, not the lowest — unclassified means irreversible-or-external-effect until proven otherwise.

### Approval Fatigue

Human approvers, facing high volume, approve without meaningfully evaluating the request.

Mitigation:

- track approval latency and approval rate per approver; a near-100% approval rate with near-instant response time is itself a signal to investigate, not a sign of a well-tuned system;
- keep the volume of human-approval-required actions bounded by keeping the risk-tier boundary narrow, not by loosening the approval requirement.

### Scope Creep at Grant Time

A task's declared scope is written broader than the task needs "to be safe" against future changes, defeating the purpose of scoping.

Mitigation:

- scope is declared per invocation, not pre-negotiated at task design time for convenience;
- audit review samples granted scope against what the task actually used, flagging systematic over-grants.

### Authorization Bypass via Direct Infrastructure Access

An agent or its host process reaches a tool's underlying system directly, bypassing the Enterprise Services authorization point entirely — the same failure class the Reference Architecture already documents as "Any Layer → Infrastructure Layer must not exist."

Mitigation:

- tool credentials are held by the Enterprise Services Layer, never distributed to the Orchestration Layer or the agent runtime itself, mirroring EDR-0002's requirement that only the routing layer holds provider credentials.

## Observability Requirements

Each tool-call authorization decision must record:

- requesting task identity and declared intent;
- tool, operation, and declared scope;
- risk classification;
- grant or denial, and the reason for denial when applicable;
- human approver identity and timestamp, for irreversible-or-external-effect actions;
- actual scope used, for post-hoc comparison against declared scope.

## Acceptance Criteria

The decision is considered successfully implemented when:

- no tool call executes without a per-task authorization record;
- every irreversible-or-external-effect action has a recorded human approval preceding execution;
- an unclassified or ambiguously classified tool defaults to denial, not to execution;
- audit review can reconstruct, for any executed tool call, the task, scope, and (if applicable) approver behind it;
- a prompt-injection red-team exercise (see the [Prompt Injection and Jailbreak Defense pattern](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md)) that successfully manipulates the model's proposed tool call is still blocked at the authorization boundary.

## Reversal Criteria

This decision should be reconsidered if:

- the operational cost of per-task authorization (latency, infrastructure, approver load) is shown to exceed the loss actually prevented, measured against real incident data rather than hypothetical risk;
- all agent workloads in scope are read-only and reversible, making risk-tiered approval unnecessary overhead;
- an equally strong enforcement mechanism is established at a different layer that makes this boundary redundant.

## Related Decisions

- [EDR-0002 — Model Routing Strategy](EDR-0002-MODEL-ROUTING-STRATEGY.md) — establishes the precedent of policy-based, centrally auditable decisions between orchestration and execution.
- [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md) — the complementary input-side control; this EDR is the blast-radius control for when that defense is bypassed.

## Review Notes

This decision must be validated against at least one runnable implementation before being marked Accepted, consistent with the review discipline already applied to EDR-0002.
