# Domain Docs

How the engineering skills should consume this repo's domain documentation when exploring the codebase.

## Before exploring, read these

- **`GLOSSARY.md`** at the repository root.
- **`docs/adr/`**: read architecture decision records that touch the area being changed.

If these files or directories do not exist yet, proceed without them. Do not flag their absence or create placeholder domain documents. Add a glossary term or ADR when a real domain distinction or durable architecture decision is resolved.

## File structure

This repository uses a single-context layout:

```text
/
├── GLOSSARY.md
├── docs/adr/
│   ├── 0001-example-decision.md
│   └── 0002-another-decision.md
└── src/
```

## Use the glossary's vocabulary

When output names a domain concept (in an issue title, refactor proposal, hypothesis, or test name), use the term as defined in `GLOSSARY.md`. Do not drift to synonyms the glossary explicitly avoids.

If the concept is not in the glossary, reconsider whether the term is project language or note the gap for `/domain-modeling`.

## Flag ADR conflicts

If a proposed change contradicts an existing ADR, surface the conflict explicitly rather than silently overriding it.
