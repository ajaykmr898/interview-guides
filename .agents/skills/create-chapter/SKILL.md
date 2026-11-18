---
name: create-chapter
description: Complete an existing interview-guides chapter from its local editorial plan. Use when asked to write or finish a supplied chapter path; not for creating new chapters.
---

# Create chapter

1. Use the supplied path to an existing chapter. If the path is missing, ambiguous, or absent from its local plan, resolve that before editing; do not create a chapter.
2. Read the repository `AGENTS.md`, the complete `EDITORIAL-PLAN.md` in the chapter's directory, the target chapter, relevant README links, and related canonical chapters.
3. Cover every mandatory **Da coprire** item, including P0, P1, and P2 topics; priority changes depth, never coverage. Preserve existing headings, anchors, navigation, and relative links.
4. Write concise definitions in Italian while retaining established English technical terms. Add the plan's required snippets and useful comparisons, best practices, limitations, and trade-offs. Give framework-specific context and link to generic explanations instead of duplicating them.
5. Verify version-sensitive claims against current official sources when available. State any claim that could not be verified.
6. Modify only the requested chapter. Check coverage, snippet syntax, Markdown links, and the final diff; report what changed and any unresolved uncertainty. Never commit or push automatically.

## Technical Completeness & Accuracy

When creating a chapter, prioritize technical accuracy and conceptual completeness over brevity.

## 1. Definitions and Core Concepts

- Define the main subject of the chapter explicitly.
- Expand acronyms at their first meaningful occurrence.
- Explain what each concept is, how it works and why it is used.
- Do not merely mention concepts without explaining them.
- Include essential characteristics, behavior and limitations.
- Preserve important technical distinctions.
- Avoid oversimplifications that could be misleading
  in a Senior-level technical interview.

## 2. API Behavior and Return Values

For each important API, method or operator:

- Verify its input and output types.
- Explain what it returns.
- Verify its actual runtime behavior.
- Distinguish synchronous from asynchronous behavior.
- Explain relevant error and failure conditions.
- Identify cancellation and cleanup behavior when applicable.
- Check important edge cases.
- Do not generalize behavior across similar APIs
  without verifying each API individually.

## 3. Comparisons

When comparing related APIs or concepts:

- Verify each item independently.
- Explain meaningful behavioral differences.
- Include return types and error behavior when relevant.
- Highlight common misconceptions.
- Do not imply that similar names mean identical behavior.

## 4. Code Examples

- Use syntactically valid and conceptually correct code.
- Ensure examples demonstrate the behavior being explained.
- Verify expected outputs and execution order.
- Do not use misleading or non-functional examples.
- Show setup and cleanup when essential.
- Avoid examples that rely on unstated assumptions.
- Prefer minimal, self-contained snippets.

## 5. Official Documentation

- Verify important and version-sensitive technical claims
  against current official documentation when accessible.
- Prefer specifications and official API references.
- Do not invent details when verification is unavailable.
- Distinguish verified facts from assumptions.
- Clearly identify experimental, deprecated or removed APIs.

Do not claim successful verification of facts or code that could not actually be checked.
Keep definitions concise, but never remove an essential detail solely to satisfy a line limit.