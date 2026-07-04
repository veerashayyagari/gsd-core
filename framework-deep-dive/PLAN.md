# GSD Core Deep Dive Plan

## Objective

Produce a readable, self-contained handbook that explains why GSD Core exists,
what surfaces it exposes, how the main workflows operate, how configuration and
capabilities affect behavior, how the system is built and installed, and where
the implementation or documentation has gaps worth improving.

The desired reader starts with only a surface-level understanding and finishes
able to critique the framework intelligently.

## Output Shape

The primary output is `GSD-CORE-DEEP-DIVE.md`. It should read like a book:

1. Start with the user-facing model.
2. Walk through the main journey.
3. Explain the artifacts created along the way.
4. Descend into routing, workflows, agents, CLI modules, configuration,
   capabilities, runtime adapters, and build output.
5. Close with strengths, gaps, and improvement opportunities.

Appendices live beside the handbook in the same folder. They are allowed to be
more table-heavy because they exist for lookup and verification.

## Source Strategy

The deep dive uses a layered source-of-truth model:

| Layer | Primary Sources |
|---|---|
| User surface | `commands/gsd/*.md`, `skills/*/SKILL.md`, `docs/INVENTORY.md`, `docs/COMMANDS.md` |
| Workflow behavior | `gsd-core/workflows/*.md`, `agents/gsd-*.md`, `gsd-core/references/*.md` |
| CLI behavior | `gsd-core/bin/gsd-tools.cjs`, `src/*command-router.cts`, `src/command-routing-hub.cts`, `src/*.cts` domain modules |
| Configuration | `gsd-core/bin/shared/config-schema.manifest.json`, `gsd-core/bin/shared/config-defaults.manifest.json`, `src/config-loader.cts`, `src/config-schema.cts`, `src/configuration.cts`, `capabilities/*/capability.json` |
| Capabilities | `capabilities/*/capability.json`, `src/capability-loader.cts`, `src/capability-state.cts`, `docs/reference/capability-*.md`, `docs/explanation/capability-*.md` |
| Runtime/build | `package.json`, `tsconfig.build.json`, `bin/install.js`, `src/runtime*.cts`, `src/install*.cts`, `docs/reference/skill-mapping-matrix.md` |
| Quality/safety | `tests/*.test.cjs`, `eslint-rules/*.cjs`, `SECURITY.md`, `docs/explanation/security-model.md` |

Broad documentation is used as explanatory context. Generated manifests and
implementation code are used to detect doc drift and hidden behavior.

## Work Plan

| Step | Status | Notes |
|---|---:|---|
| Create isolated folder | Done | All artifacts live under `framework-deep-dive/`. |
| Extract command/skill/capability/config inventories | Done | Counts: 70 commands, 70 skills, 34 agents, 90 workflows, 33 capabilities. |
| Draft handbook narrative | Done | Main document covers surface -> internals -> critique. |
| Draft skill atlas | Done | High-level summaries avoid requiring readers to open every `SKILL.md`. |
| Draft configuration catalog | Done | Distinguishes central valid keys, dynamic key patterns, and capability-owned federated keys. |
| Draft flow traces | Done | Core phase loop, install, config, capability, verify/ship. |
| Validate doc claims against source | Done | Acceptance validation is recorded in `VALIDATION.md`; count claims and catalogs were checked against the checkout. |
| Commit docs | Pending | Commit should include only this new folder after validation is accepted. |

## Acceptance Criteria

The deep dive is useful when:

- A reader can explain the GSD Core mental model without reading source files.
- A reader can follow the core loop from skill invocation to workflow, agents,
  CLI tools, and `.planning/` artifacts.
- Configuration options are explained from code-discovered truth, including
  code-only or capability-owned keys.
- Every skill has a high-level summary with enough context to decide whether to
  drill down.
- The capability system is explained as runtime extension, surface control,
  config federation, and loop hook composition.
- The build/install story explains authored `src/*.cts`, generated
  `gsd-core/bin/lib/*.cjs`, package files, runtime artifact conversion, and
  runtime-specific layouts.
- Gaps are captured as actionable critique, not just observations.

## Open Questions For Later Iterations

- Which config keys are intentionally hidden power-user settings versus
  accidentally undocumented behavior?
- Should the public docs promote the capability-owned config catalog, or keep it
  in generated reference only?
- Should the skill atlas be generated as part of release validation to prevent
  drift?
- Should config keys carry ownership metadata directly in the schema manifest so
  consumers do not need to infer ownership from capability registries and usage?
