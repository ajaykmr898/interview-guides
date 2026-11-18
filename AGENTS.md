# AGENTS.md — Interview Guides

## Project Context

This repository contains concise technical interview revision guides.

Primary audience:
- Middle, Senior and Expert software developers.
- Developers preparing for technical interviews.
- Developers refreshing concepts they already know.

The documentation is NOT a beginner tutorial or textbook.

Goals:
- Accurate technical definitions.
- Fast revision and easy navigation.
- Comprehensive coverage from fundamentals to advanced topics.
- Short, useful code examples.
- Best practices and technical trade-offs.

All guide content MUST be written in Italian.
Keep established English technical terminology when appropriate.

## Scope and Ownership

Keep each technology's chapter requirements in its local `EDITORIAL-PLAN.md`. Preserve the repository structure and work only on the technologies and chapters requested for the current task. Create additional chapters only when explicitly requested.

When a concept has a canonical chapter elsewhere, explain the local context briefly and use a relative Markdown link to the detailed definition. Keep technology-specific APIs and examples in their own chapters.

## Editorial Standards

Write in concise, professional Italian.

Definition length:
- Simple concepts: 1–2 lines.
- Standard concepts: 2–4 lines.
- Complex concepts: 4–6 lines.

### Local editorial plans

Before creating or editing a chapter, read this file and the complete `EDITORIAL-PLAN.md` in that chapter's technology directory. The local plan defines required concepts; this file defines shared writing and validation rules. If a local plan is missing, report the gap instead of inventing chapter requirements.

The local plan's **Da coprire** entries are mandatory coverage, not prescribed headings. Each concept needs an identifiable explanation, not just a name in a list.

Coverage priorities determine depth, never permission to omit a listed concept:

- **P0 — Essential:** include definition, behavior and essential distinctions.
- **P1 — Important:** include at least definition, use cases and main limitations.
- **P2 — Advanced or specialist:** explain briefly and link related concepts already covered.

### Concept format

For each relevant concept, use a heading at the appropriate level and these elements when useful:

**Definizione:** A precise technical explanation.

**Punti chiave:**
- Important behavior or property.
- Relevant limitation or distinction.
- Additional detail only when useful.

**Esempio:** A minimal code snippet, when appropriate.

**Best practice:** A practical recommendation when relevant.

**Differenze:** Compare related concepts when useful.

Do not mechanically include every element for every concept.

Additional rules:
- Prefer bullet points over long paragraphs.
- Use Markdown tables for technical comparisons.
- Keep code examples generally between 3 and 10 lines.
- Use proper syntax highlighting in fenced code blocks.
- Explain why and when a technique is useful.
- Avoid unnecessary introductions and conclusions.
- Avoid motivational text and conversational filler.
- Do not include quizzes, challenges or exercises.

## Content Coverage

Cover concepts from Junior through Expert level.

For every chapter:
- Include all important concepts within its defined scope.
- Do not sacrifice important definitions for brevity.
- Prioritize frequently encountered interview concepts.
- Include advanced concepts, limitations and trade-offs.
- Explain technical terminology accurately.
- Include legacy approaches when relevant.
- Distinguish legacy, current, deprecated and removed APIs.

Do not create superficial summaries that merely list concept names.

The reader should be able to understand and explain a concept after reading its definition.

## Technical Accuracy

Technical correctness has priority over brevity.

Rules:
- Verify technical facts and version-sensitive information against current official documentation for the relevant technology.
- Never invent API names, syntax, behavior or release dates.
- When version status matters, distinguish introduced, stable, default, experimental, deprecated and removed APIs.
- Ensure code snippets use valid syntax.
- Explain important edge cases and limitations.
- Do not present recommendations as universal rules.
- State uncertainty when something cannot be verified.

Avoid unnecessary external links in the main explanations.

## Markdown and Navigation Rules

When editing:
- Preserve existing navigation, heading names and anchors.
- Use consistent heading hierarchy.
- Keep relative links valid.
- Avoid unnecessary restructuring.
- Do not rewrite completed sections without a reason.

## Working Procedure

Process one chapter at a time unless instructed otherwise.

Workflow:
1. Read the target chapter when it exists and relevant README links.
2. Identify all required concepts in the local plan and their canonical sources.
3. Check related shared chapters to avoid duplication.
4. Verify technical facts when necessary.
5. Write concise definitions and examples.
6. Review completeness and accuracy.
7. Validate Markdown structure and links.
8. Inspect the final diff.

Do not install dependencies or create build tools.
Do not create unrelated files.
Do not commit or push changes unless explicitly requested.

## Final Quality Checklist

Before reporting completion, verify:

- Content is in Italian technical language.
- All requested concepts are covered.
- Definitions are concise and accurate.
- Examples are minimal and correct.
- Relevant comparisons are present.
- Best practices include appropriate context.
- No unnecessary content duplication exists.
- Existing headings and links are preserved.
- The file contains no quizzes, simulated exams, code challenges or exercises.
- Version-sensitive claims have been verified.
- Every **Da coprire** entry in the local editorial plan has an identifiable explanation.
- Framework-specific snippets are in the appropriate chapter; comparisons cover easily confused concepts.
- The internal index remains consistent, and relative links work.

Report:
- Files modified.
- Main sections completed.
- Any unresolved technical uncertainties.
- Any risks or facts that could not be verified.
- Any missing topics requiring future work.
