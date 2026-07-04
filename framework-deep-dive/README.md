# GSD Core Deep Dive

This folder is intentionally isolated from the product source and existing
documentation. It is a self-contained analysis workspace for understanding GSD
Core as a framework, from the user-facing surface down to implementation,
configuration, build, runtime, and critique points.

Start here:

1. Read `GSD-CORE-DEEP-DIVE.md` like a book. It is written so a reader does not
   need to jump between files to build the mental model.
2. Use `PLAN.md` to understand the research and writing workflow behind the
   handbook.
3. Use `appendices/` only when you want complete catalogs or trace evidence.

## Artifacts

| File | Purpose |
|---|---|
| `PLAN.md` | Execution plan, status, source strategy, and acceptance criteria for this deep dive. |
| `VALIDATION.md` | Pre-commit validation against the plan acceptance criteria and source-derived coverage checks. |
| `GSD-CORE-DEEP-DIVE.md` | Main readable handbook. Starts at the surface and descends into internals. |
| `appendices/SKILL-ATLAS.md` | High-level catalog of every shipped skill and namespace router. |
| `appendices/CONFIGURATION-CATALOG.md` | Central, dynamic, and capability-owned configuration catalog. |
| `appendices/FLOW-TRACES.md` | End-to-end traces for the main framework journeys. |
| `appendices/SOURCE-OF-TRUTH-MATRIX.md` | Which files are authoritative for each framework concern. |
| `appendices/GAP-REGISTER.md` | Initial critique register for documentation, code, tests, and maintainability gaps. |

## Ground Rules

- Treat code and generated manifests as source of truth.
- Treat broad docs as explanation to verify against source, not as authority.
- Keep the main handbook readable and narrative.
- Put full catalogs in appendices, but summarize their meaning in the handbook.
- Keep all future deep-dive edits inside this folder unless deliberately
  promoted into product documentation.
