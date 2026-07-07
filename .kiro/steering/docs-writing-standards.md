---
inclusion: always
---

# Documentation Writing Standards — Stellar Guardian

## Purpose
This steering file defines the writing conventions, quality bar, tone, and structural standards for all documentation produced in this project. Every document must conform to these standards before being considered complete.

---

## 1. Tone & Voice

- **Professional but direct.** No filler phrases, no marketing fluff.
- **Engineering-first.** Assume the reader is a senior engineer or technical product manager.
- **Precise.** Prefer specific numbers, named technologies, and concrete decisions over vague generalizations.
- **Active voice.** "The contract releases funds" — not "Funds are released by the contract."
- **No hyperbole.** Do not use words like "revolutionary", "world-class", "cutting-edge", or "industry-leading".

---

## 2. Document Structure

Every document must follow this structure:

```
# [Document Title]

> **Status:** Draft | Review | Approved
> **Version:** X.Y
> **Last Updated:** YYYY-MM-DD
> **Owner:** [Team or Person]

## Table of Contents
(auto-generated or manually listed)

## [Section 1]
...
## [Section N]
...

---
*[Document classification and revision notes at bottom]*
```

---

## 3. Markdown Formatting Rules

- Use `##` for major sections, `###` for subsections, `####` for deep detail.
- Use **bold** for key terms on first use, not for general emphasis.
- Use `code blocks` for: file paths, identifiers, commands, contract names, SQL, code snippets.
- Use tables for comparison matrices, requirement mappings, and structured data.
- Use blockquotes (`>`) for callouts, warnings, and design rationale notes.
- Use numbered lists for sequential steps. Use bullet lists for non-ordered items.
- Avoid nested bullets deeper than 2 levels.

---

## 4. Diagram Standards

- Use **Mermaid** syntax for all diagrams embedded in Markdown.
- Supported diagram types: `flowchart`, `classDiagram`, `erDiagram`, `sequenceDiagram`, `C4Context`.
- Every diagram must have a caption immediately below it.
- ASCII art diagrams from the PDF source are acceptable as fallback when Mermaid is insufficient.

---

## 5. Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Document files | `NN_SCREAMING_SNAKE.md` | `01_PRODUCT_DISCOVERY.md` |
| Contract names | `PascalCase` | `EscrowAgreement` |
| Rust types | `snake_case` | `escrow_core` |
| SQL tables | `snake_case` | `escrow_milestones` |
| API endpoints | `kebab-case` | `/api/v1/escrows` |
| Environment vars | `SCREAMING_SNAKE_CASE` | `STELLAR_RPC_URL` |

---

## 6. Cross-Reference Standards

- When referencing another document in the `docs/` folder, use a relative markdown link.
- When referencing a Stellar Ecosystem Proposal (SEP), always link to the official spec: `https://github.com/stellar/stellar-protocol/blob/master/ecosystem/`
- When referencing a CVE, always include both the CVE ID and a one-line description.
- Do not use citation numbers like `[cite: 7]` in final documents — convert them to inline prose references or footnotes.

---

## 7. Completeness Checklist

Before marking any document as `Approved`, verify:

- [ ] All sections have content (no `TODO` placeholders)
- [ ] All diagrams render correctly
- [ ] All decisions include rationale (not just what, but why)
- [ ] All acronyms are defined on first use
- [ ] Version, status, and date metadata are filled in
- [ ] Cross-references to related documents are present

---

## 8. Document Series Order

The canonical documentation order for this project is:

```
01_PRODUCT_DISCOVERY.md
02_REQUIREMENTS_SRS.md
03_ARCHITECTURE_REVIEW.md
04_ENGINEERING_BLUEPRINT.md
05_DOMAIN_MODEL.md
06_DATABASE_DESIGN.md
07_API_SPECIFICATION.md
08_SMART_CONTRACT_SPEC.md
09_UI_UX_SYSTEM.md
10_SECURITY.md
11_TESTING_STRATEGY.md
12_DEVOPS.md
13_AGILE_ROADMAP.md
14_IMPLEMENTATION_GUIDE.md
```

Each document builds on the previous. Do not create a later document if an earlier one is incomplete.
