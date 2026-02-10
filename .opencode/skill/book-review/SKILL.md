---
name: book-review
description: Use this when an official maintainer reviews a tutorial or guide written for their own open-source project. This skill ensures technical accuracy, conceptual clarity, and alignment with both the current implementation and the project’s intended design philosophy.
---

## Use this when

- Writing an official tutorial or guide for your own open-source project
- Reviewing Markdown-based learning materials stored inside the repository
- Teaching users how to understand, extend, or contribute to the project
- Validating that explanations reflect both how the system works and why
- Preparing content that will be treated as canonical by the community

This skill assumes **authorial authority over the codebase**.

---

## Official-tutorial review dimensions

### Implementation accuracy (authoritative)

- Verify that described behavior matches the current implementation
- Flag explanations that contradict real runtime behavior
- Identify places where implementation details are intentionally abstracted
- Ensure code examples would work against the actual codebase

### Design intent & philosophy

- Check that tutorials communicate the project’s core design principles
- Identify explanations that reflect accidents of implementation rather than intent
- Flag places where design trade-offs should be made explicit
- Ensure abstractions are framed as deliberate, not incidental

### Evolution-aware narrative

- Identify features or patterns that are transitional or legacy
- Check whether historical context is needed to avoid confusion
- Flag tutorials that freeze the project in its current state
- Ensure forward-looking statements are clearly marked as such

### Learning-path coherence

- Verify that concepts are introduced in an order aligned with how users learn
- Detect cognitive jumps that rely on maintainer-only knowledge
- Identify places where internal complexity leaks too early
- Flag sections where examples assume undocumented prerequisites

### Community-facing precision

- Check that terminology matches what contributors will see in code and issues
- Ensure naming is consistent with public APIs and docs
- Flag ambiguity that could generate recurring user questions
- Identify statements likely to be over-interpreted as guarantees

---

## Output format

All findings must follow this exact structure:

- 【Issue Type】
- 【Excerpt (Markdown)】
- 【Code / API Reference】
- 【Explanation】
- 【Suggestion (Optional)】

Rules:
- Do not rewrite the tutorial
- Do not refactor code as part of review
- Do not assume reader intent beyond stated goals
- Distinguish clearly between “how it works” and “why it is designed this way”

---

## Skill constraints

- Treat the author as the ultimate authority on intent
- Treat the codebase as the authority on behavior
- Surface tension between intent and behavior instead of resolving it silently
- Avoid over-specifying future guarantees
- Silence is valid when content is both accurate and appropriate

---

## Official-author patterns

- Tutorials define the public mental model of the project
- Readers will treat examples as normative
- Design explanations shape future contributions
- Precision prevents long-term community confusion
- Authority increases responsibility, not license

---

## Quick checklist

- Would users treat this as canonical guidance?
- Does this reflect how the project actually behaves today?
- Is design intent explicit where behavior is non-obvious?
- Are future directions clearly labeled?
- Did I review as a maintainer, not as an external critic?
