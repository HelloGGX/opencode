---
name: book-authoring
description: Use this when writing a technical book that teaches readers to recreate a real-world open-source project from 0 to 1. This skill enforces decision-driven storytelling and guided learning through code, focusing on why designs were chosen and how alternatives fail.
---

## Use this when

- Writing an authoritative technical book based on a real codebase
- Teaching readers to rebuild a system step by step
- Emphasizing engineering judgment over API memorization
- Using code as a primary explanatory medium, not as reference material
- Guiding readers to think like engineers, not just follow instructions

This skill assumes the author teaches by **revealing decisions**, not by dumping conclusions.

---

## Core information to extract (before writing)

### System blueprint (macro view)

- Repository file tree (excluding irrelevant directories)
- Dependency manifests as evidence of technical choices
- Clear entry points and one end-to-end “golden path”
- Architectural boundaries and seams

Purpose:  
Establish the **structural and narrative backbone** of the book.

---

### Core logic slices (micro view)

- Domain models and type definitions expressing business intent
- High-reuse or high-complexity utilities and algorithms
- Tests as executable specifications and behavioral contracts
- Invariants and constraints enforced by code

Purpose:  
Define the **technical ground truth** that explanations must respect.

---

### Evolution & decision context (senior-engineer view)

- Commit history or blame to infer implementation sequence
- Evidence of refactors, reversals, or rejected approaches
- README, design docs, and contributing guidelines
- Edge cases revealed through tests and bug fixes

Purpose:  
Recover the **decision trail hidden behind the final implementation**.

---

## Authoring methodology (Reverse-Engineering Pedagogy)

### Step 1: Architecture-first outlining

- Derive chapter order from dependency and build order, not file order
- Each chapter represents a *valid intermediate system state*
- No chapter assumes mechanisms that have not yet been built
- Each chapter ends with a clear milestone

Guiding question:  
> “If the reader stopped here, would they understand *why* the system looks this way so far?”

---

### Step 2: Simulated iterative development

- Introduce a naive or minimal solution first
- Clearly expose its limitations in real-world scenarios
- Use those limitations to *force* the next design decision
- Present the final implementation as a response, not a revelation

Mandatory framing:
- What problem emerged?
- Why the naive solution fails?
- What trade-off does the chosen design accept?

---

### Step 3: Code as narrative, not artifact (关键增强点)

Code is treated as the **primary explanatory language**, not supplementary material.

Rules:
- Code snippets must be minimal and focused on the current teaching point
- Every snippet must exist to answer a specific question raised by the narrative
- Prefer **before / after** comparisons to show evolution and optimization
- Never present code in isolation from its decision context

Required explanation structure for each code segment:
- **Context** — where this code sits in the system
- **Problem** — what concrete issue forced this code to exist
- **Insight** — what design principle or trade-off it embodies

Prohibited patterns:
- Line-by-line narration
- Full-file dumps without justification
- Treating code as self-explanatory
- Introducing multiple ideas in a single snippet

---

## Guided learning style (写作风格约束)

### From instruction to guidance

- Do not tell readers what to think immediately
- Ask the question *before* giving the answer
- Let readers anticipate the problem before revealing the solution
- Use code changes to confirm or falsify intuition

### From delivery to discovery

- Position the reader as an active problem-solver
- Walk alongside the reader’s reasoning process
- Make trade-offs explicit and unavoidable
- Optimize for “aha” moments, not information density

Core principle:
> **Teach readers how to arrive at decisions, not how to copy them.**

---

## Authoring constraints

- Do not present final code without showing its necessity
- Do not collapse multiple decisions into a single explanation
- Do not hide complexity without naming its cost
- Do not optimize for brevity at the expense of insight
- Do not write from a god’s-eye view of the finished system

---

## Book design principles (senior-level)

- Code is part of the argument, not an appendix
- Design decisions are first-class teaching material
- Trade-offs are more important than correctness alone
- Tests and failures are narrative assets
- Mastery comes from reasoning, not repetition

---

## Quick checklist (before finalizing a section)

- Did I raise the question before answering it?
- Does the code *prove* the point I am making?
- Can the reader see why earlier versions failed?
- Is the reader thinking with me, not following me?
- Am I teaching judgment rather than syntax?
