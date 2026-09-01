# Architectural Decision Records

This folder holds Architectural Decision Records (ADRs) for this project — short documents that capture a real design decision and the trade-offs behind it, written down so the reasoning doesn't have to be reverse-engineered from the code later.

## Naming and numbering

Each ADR is a single Markdown file named `NNNN-kebab-title.md`, where `NNNN` is a sequential 4-digit prefix (`0001`, `0002`, …) and the title is a short kebab-case summary of the decision. Numbers are assigned in order and never reused. See the existing files for worked examples:

- `0001-generic-account-model.md`
- `0002-fx-rate-point-in-time.md`
- `0003-category-rules-additive-scoped-recompute.md`

## Content shape

There's no fixed template with required headings — each ADR is free-flowing prose: a title stating the decision, followed by one or more paragraphs covering the decision itself, the rationale, and the trade-offs or alternatives considered. Keep it descriptive of the actual decision rather than filling in a generic checklist.

## When to add one

Add a new ADR for a real decision with meaningful trade-offs — something a future reader would otherwise have to guess at or reverse-engineer from the code. Routine implementation details that don't involve a trade-off worth recording don't need one; this project keeps its process low-overhead by design (see `0001-generic-account-model.md`).
