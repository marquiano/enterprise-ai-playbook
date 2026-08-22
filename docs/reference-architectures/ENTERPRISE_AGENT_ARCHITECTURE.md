# Enterprise Agent Architecture

> Reference Architecture — the internal structure of an agentic system operating inside the AI Orchestration Layer of the [Enterprise AI Reference Architecture](ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md).

---

## Relationship to the Enterprise AI Reference Architecture

An "agent" — a system that plans across multiple steps and invokes tools to accomplish a task, rather than producing a single response — lives inside the AI Orchestration Layer. It does not introduce a new layer; it is a specific internal structure that layer can take on. Everything the main Reference Architecture already decided still applies: the agent calls Enterprise Services with declared scope, Enterprise Services authorizes independently, and no path from the agent reaches Infrastructure directly.

This document defines what "AI Orchestration Layer" means specifically when that layer is agentic, and does not restate what the main Reference Architecture already covers.

## Components

### Planner

Decomposes a task into a sequence of steps, decides the next action given current state, and determines when the task is complete. The planner is the component most exposed to [prompt injection](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md), since its input includes not just the original request but the results of every prior step — each of which is untrusted content by the pattern's own definition.

### Executor

Carries out a single planned step, typically a tool invocation. The executor is the component that requests authorization from Enterprise Services per [EDR-0003](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md) — it proposes a call; it does not self-authorize.

### State/Memory Store

Holds the task's working state across steps: prior actions, tool results, and intermediate conclusions. State that persists beyond a single task execution (long-term memory, cross-session context) is a distinct trust boundary from within-task state — see Contracts below.

### Delegation Boundary

Where a task hands part of its work to another agent (sub-agent, specialized agent, or a different autonomy tier). Delegation is a request for a new authorization, not an inheritance of the delegating agent's grants — see Contracts below.

## Contracts

### Planner → Executor

**Provides:** a single, concrete next action with its declared scope — not an open-ended goal the executor interprets on its own.

**Guarantees:** the executor treats the planner's proposed action as a proposal, consistent with the main Reference Architecture's Orchestration → Enterprise Services contract — the executor does not expand scope beyond what the planner specified, and Enterprise Services authorizes independently of both.

### Executor → State/Memory Store

**Provides:** the result of each executed step, written to within-task working state.

**Guarantees:** within-task state is scoped to the current task and does not silently persist into a future task's context — an agent starting a new task does not inherit a prior task's state unless that persistence is an explicit, declared design choice with its own data-sensitivity classification, per the main Reference Architecture's Security Requirements.

### Delegating Agent → Sub-Agent

**Provides:** a task-scoped request with its own declared intent — not a copy of the delegating agent's full permission grant.

**Guarantees:** the sub-agent's tool authorizations are requested fresh against [EDR-0003](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md), scoped to the delegated task. A sub-agent is never granted broader scope than its specific delegated task requires, even if the delegating agent holds broader scope for its own task.

### Planner → Task Completion

**Provides:** an explicit termination condition — the planner declares the task complete, or declares it cannot proceed.

**Guarantees:** a task has a bounded step count and/or bounded resource budget declared before execution begins; there is no unbounded "keep trying" default.

## Autonomy Tiers

Autonomy tiers map directly onto the risk tiers EDR-0003 already defines for tool authorization — this document does not invent a parallel scale.

| Tier | Planner Behavior | Executor Behavior |
|---|---|---|
| Read-only | Plans and executes freely within read-only tool scope | No human approval required (matches EDR-0003 read-only tier) |
| Reversible-write | Plans freely; each write step requests authorization | No human approval required per step, but scope and volume are monitored (matches EDR-0003 reversible-write tier) |
| Irreversible-or-external-effect | Plans up to, but does not execute, the irreversible step without a human checkpoint | Requires human approval before execution, per EDR-0003 |

A task that mixes tiers (e.g., mostly read-only with one irreversible step near the end) is not elevated wholesale to the highest tier for its entire execution — only the specific step at that tier requires the corresponding control. Treating an entire task as high-risk because one step is defeats the Decision Driver in EDR-0003 that scope should match what the task actually requires, and reintroduces the approval-fatigue failure mode EDR-0003 already documents.

## Failure Paths

### Goal Drift Across Steps

**Cause:** the planner's understanding of the task shifts over the course of many steps — either through accumulated small reasoning errors or through a successful prompt injection at an intermediate step — so that later steps no longer serve the original request.

**Detection:** compare each proposed step's declared intent against the original task's declared intent; a step whose intent diverges beyond a defined similarity threshold is flagged before execution, not after.

**Mitigation:** re-anchor the planner to the original task statement at a fixed step interval, not only at task start; treat a flagged divergence the same as an unauthorized-scope tool call under EDR-0003 — pause for authorization rather than proceeding.

### Runaway Execution Loop

**Cause:** the planner repeatedly proposes similar or identical steps without making progress toward task completion — a stuck loop, whether from a reasoning failure or a tool that returns results the planner cannot productively act on.

**Detection:** step count and elapsed-resource tracking against the task's declared budget (per the Planner → Task Completion contract above); repeated near-identical proposed actions within a short window.

**Mitigation:** hard step and resource budgets terminate the task rather than allowing indefinite continuation; a terminated task is reported as incomplete, not silently retried without limit.

### Sub-Agent Scope Inheritance

**Cause:** a delegating agent's implementation passes its own tool credentials or authorization context directly to a sub-agent, rather than requesting a fresh, narrower authorization for the delegated task — violating the Delegating Agent → Sub-Agent contract above.

**Detection:** audit trail (per EDR-0003's Observability Requirements) shows a sub-agent's executed scope matching the delegating agent's full grant rather than a narrower, task-specific one.

**Mitigation:** sub-agent authorization is always requested fresh against Enterprise Services; credentials are never held by the agent runtime layer in the first place, per the main Reference Architecture's Orchestration → Enterprise Services contract.

## Related Artifacts

- [Enterprise AI Reference Architecture](ENTERPRISE_AI_REFERENCE_ARCHITECTURE.md)
- [EDR-0003 — Agent Tool-Scope and Permission Boundary](../engineering-decisions/EDR-0003-AGENT-TOOL-PERMISSION-BOUNDARY.md)
- [Prompt Injection and Jailbreak Defense](../engineering-patterns/PROMPT_INJECTION_AND_JAILBREAK_DEFENSE.md)
- [Enterprise AI Evaluation Strategy](../evaluation/ENTERPRISE_AI_EVALUATION_STRATEGY.md)
- [AI Incident and Rollback Playbook](../incident-playbooks/AI_INCIDENT_AND_ROLLBACK_PLAYBOOK.md)
