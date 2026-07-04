# Source Of Truth Matrix

This matrix records where a reader should look first when trying to understand
or change a framework concern.

| Concern | Source Of Truth | Supporting Docs / Tests | Notes |
|---|---|---|---|
| Package identity and bins | `package.json` | `tests/package-manifest.test.cjs`, package identity tests | Exposes `gsd-core`, `gsd-tools`, `gsd_run`, and `gsd-mcp-server`. |
| TypeScript build | `tsconfig.build.json`, `src/*.cts` | `scripts/run-tests.cjs`, ADR-457 docs | Authored `.cts` emits generated `.cjs` to `gsd-core/bin/lib/`. |
| Runtime CJS modules | `gsd-core/bin/lib/*.cjs` | Build/test runner | Generated runtime output. Edit `src/*.cts` when a source file exists. |
| Commands | `commands/gsd/*.md` | `docs/INVENTORY.md`, `docs/COMMANDS.md`, command parity tests | User-facing slash command source and skill conversion source. |
| Skills | `skills/*/SKILL.md` | `docs/INVENTORY.md`, skill frontmatter tests | Installed/generated skill surface. Some runtimes emit from command sources. |
| Workflows | `gsd-core/workflows/*.md` | workflow size tests, docs inventory | Prompt orchestration logic. |
| Agents | `agents/gsd-*.md` | `docs/AGENTS.md`, agent tests | Specialized subagent definitions and tool permissions. |
| References | `gsd-core/references/*.md` | inventory and prompt-size tests | Shared knowledge for workflows and agents. |
| Planning artifacts | `gsd-core/templates/*.md`, docs reference schemas | `docs/reference/*-md.md`, planning artifact tests | Templates define expected persisted project memory. |
| Central config keys | `gsd-core/bin/shared/config-schema.manifest.json` | `src/config-schema.cts`, config schema tests | 100 central valid keys in current checkout. |
| Config defaults | `gsd-core/bin/shared/config-defaults.manifest.json` | `src/configuration.cts`, install tests | Defaults are nested canonical shape. Some legacy flat projections remain. |
| Config loading | `src/config-loader.cts` | config loader tests, loop render hook tests | Merges defaults, root config, workstream config, legacy normalizations, and federated capability config. |
| Dynamic config keys | `config-schema.manifest.json` dynamic patterns | config schema tests | Allows controlled key families such as `models.*` and `review.models.*`. |
| Capability-owned config | `capabilities/*/capability.json` | `src/federated-config.cts`, capability loader tests | Federates into valid config when registry is loaded. |
| Capability registry | `capabilities/*/capability.json`, generator scripts, generated `capability-registry.cjs` | `docs/reference/capability-matrix.md`, capability tests | First-party capabilities are generated into the frozen registry. |
| Third-party capability overlay | `src/capability-loader.cts`, `src/capability-consent.cts`, `src/capability-ledger.cts` | capability trust docs/tests | Overlay load is lazy, cwd-aware, consent-bound, and fail-closed for skipped gates. |
| Capability lifecycle/write paths | `src/capability-lifecycle.cts`, `src/capability-writer.cts`, `src/capability-state.cts` | capability command tests | Installs, removes, ledgers, and state toggles have distinct code paths. |
| Runtime artifact layout | `src/runtime-artifact-layout.cts`, `src/runtime-artifact-conversion.cts`, runtime descriptors | `docs/reference/skill-mapping-matrix.md` | Determines flat/nested skills, commands, hooks, and settings per host. |
| Installer | `bin/install.js`, `src/install-engine.cts`, `src/install-profiles.cts` | install tests | Emits runtime-specific surfaces and shared runtime files. `bin/install.js` still owns important edge orchestration. |
| Command routing | `src/command-routing-hub.cts`, `src/*command-router.cts`, `gsd-core/bin/gsd-tools.cjs` | command router tests | Hub returns typed results and centralizes no-throw dispatch semantics. |
| Hooks | `hooks/`, `hooks/lib/`, hook build scripts, capability loop hooks | hook tests, context monitor docs | Runtime hook support differs by host. |
| Test harness | `scripts/run-tests.cjs` | `tests/run-tests-harness.test.cjs` | Ensures generated lib and hook dist artifacts before running tests. |
| Lint policy | `eslint-rules/*.cjs`, `eslint.config.mjs` | rule tests | Enforces portability, markdown parsing, test rigor, and file operation policies. |
