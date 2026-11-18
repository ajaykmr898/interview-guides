---
name: review-chapter
description: Review an existing interview-guides chapter against repository rules and its local editorial plan. Use for a read-only quality review of a supplied chapter path; not for editing it.
---

# Review chapter

1. Use the supplied path to an existing chapter. Read `AGENTS.md`, the complete local `EDITORIAL-PLAN.md`, the target chapter, and relevant README and canonical chapters.
2. Check every required plan item, including P0, P1, and P2 coverage. Check factual accuracy, code snippet syntax, relative links and anchors, Markdown structure, duplication, comparisons, and version-sensitive API status against official sources when available.
3. Classify each finding as **Critical** (materially wrong or unsafe), **Major** (missing required coverage, broken example or navigation), or **Minor** (clarity or formatting). Mark it **Verified** only with supporting evidence; otherwise mark it **Potential** and name the needed check.
4. Report findings concisely with chapter file and line references, evidence or uncertainty, and a suggested correction. State explicitly when there are no verified findings and note any review limits.

This is a read-only review. Do not modify any files.

## Technical Accuracy Review

- Do not merely check whether a topic is mentioned.
- Verify that technical definitions and explanations are accurate.
- Validate code snippets for correctness and expected behavior.
- Check edge cases, actual runtime behavior and API limitations.
- Identify misleading simplifications or incorrect assumptions.
- Verify version-sensitive claims against official documentation.
- Distinguish confirmed errors from potential issues.
- Report inaccuracies with clear explanations and suggested fixes.