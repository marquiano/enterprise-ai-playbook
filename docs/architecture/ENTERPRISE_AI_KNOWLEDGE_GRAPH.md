# Enterprise AI Knowledge Graph

> Canonical map of the Enterprise AI Playbook.

---

## Overview

The Enterprise AI Playbook is organized as an engineering knowledge graph.

Rather than treating documentation as isolated topics, each document contributes to a connected body of engineering knowledge.

---

```mermaid
flowchart TD

EAI["Enterprise AI Engineering"]

EAI --> P["Principles"]
EAI --> L["Lifecycle"]
EAI --> RA["Reference Architectures"]
EAI --> RM["Reference Models"]
EAI --> TD["Technology Decisions"]
EAI --> DG["Deployment Guides"]
EAI --> GOV["Governance"]
EAI --> SEC["Security"]
EAI --> OPS["Operations"]
EAI --> CS["Case Studies"]

P --> L

L --> RA
L --> TD
L --> DG
L --> OPS

RA --> GOV
RA --> SEC
RA --> DG

RM --> L

TD --> DG

DG --> OPS

OPS --> CS

CS --> RM
```

---

# Reading Order

Recommended sequence for first-time readers:

1. Enterprise AI Principles
2. Enterprise AI Lifecycle
3. Enterprise AI Reference Architecture
4. Enterprise AI Maturity Model
5. Technology Decisions
6. Deployment Guides
7. Operations
8. Case Studies

---

# Engineering Philosophy

Knowledge is cumulative.

Each document should:

- extend previous knowledge;
- avoid duplication;
- reference related concepts;
- remain technology-agnostic whenever possible.