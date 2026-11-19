---
name: scaffold-technology
description: Scaffold a requested interview-guides technology path as one topic chapter or a compact framework guide with chapter outlines, editorial plan and navigation. Use before full chapter writing.
---

# Scaffold technology

## Invocation

Supply one explicit path inside the repository (relative or absolute) and, when useful, a short scope:

- `$scaffold-technology topics/api/16-restful-apis.md` — scaffold one topic chapter and its local editorial plan.
- `$scaffold-technology guides/react` — scaffold a compact React guide with chapter files, README and editorial plan.

Resolve an ambiguous target before creating files. Preserve existing populated files; add focused entries to an existing plan or README, and never replace their content without explicit permission.

## Workflow

1. Read `AGENTS.md`; inspect `topics/`, `guides/`, the root README, nearby chapters and complete local editorial plans. Identify naming, numbering, Markdown and navigation conventions, plus generic topics that already own related concepts.
2. Before writing, identify the intended files and present a compact outline: chapter filenames, descriptive titles, H2/H3 sections and each chapter's **P0/P1/P2** coverage. Choose a technology-specific breakdown for Mid/Senior/Expert interviews; combine related material rather than creating chapters merely to increase coverage. For an applicable framework or library guide, include the TodoList and ecosystem chapter types defined in `AGENTS.md` by default, unless the requested scope excludes them.
3. For `topics/<technology>/<chapter>.md`, create the requested focused chapter scaffold and a local `EDITORIAL-PLAN.md` (or add its entry to an existing plan). For `guides/<technology>`, create the compact set of numbered chapter scaffolds, a README linking every chapter, and the local plan. Keep existing populated chapters intact.
4. In every new chapter, write an H1 title, `## Indice` with links to its H2 sections, and logical H2/H3 headings with important technology-specific subsections. Use descriptive filenames and headings; omit full explanations, snippets, filler and unrelated generic sections.
5. In the plan, map mandatory **Da coprire** concepts to every chapter under P0 (essential behavior and distinctions), P1 (use cases and limits) and P2 (specialist topics). Record scope boundaries and relative links to canonical generic topics instead of duplicating their definitions. Keep all guide-facing text in concise Italian with established English technical terms.
6. Add the technology to the root README where appropriate and keep local navigation consistent. Verify that every planned chapter exists, each TOC anchor and relative link resolves, priorities cover the planned headings, existing content remains intact, and the diff contains only the requested technology and necessary navigation edits. Report files and unresolved decisions. Leave full writing to `create-chapter` and full reviews to `review-chapter`; do not add agents, tools, skills or dependencies, or commit or push.
