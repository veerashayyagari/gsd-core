# Gap Register

This register captures initial critique points discovered while building the
deep dive. It is intentionally evidence-oriented: each gap should be traceable
to a source file, manifest, or reader experience.

## High Priority

| Gap | Evidence | Why It Matters | Suggested Improvement |
|---|---|---|---|
| Configuration is distributed across too many sources. | Central schema in `gsd-core/bin/shared/config-schema.manifest.json`, defaults in `config-defaults.manifest.json`, behavior in `src/config-loader.cts`, capability keys in `capabilities/*/capability.json`, explanation in `docs/CONFIGURATION.md`. | A user or contributor cannot understand effective config from one place. | Generate a unified config reference from central schema plus capability manifests and include ownership/default/doc status. |
| Capability-owned config keys are not obvious from the central schema. | Keys such as `mempalace.enabled`, `external_job.enabled`, `workflow.schema_push_detection`, `workflow.assumption_delta` are declared by capabilities. | Readers may think a key is invalid or miss important behavior. | Add "central", "dynamic", "capability-owned", and "runtime-state" ownership columns to generated docs. |
| Plan-phase behavior is capability-dense but not easily visualized. | `plan:pre` and `plan:post` hooks include research, UI, pattern mapper, intel, MemPalace, schema, drift, security, TDD, gap analysis, and external job. | Users may not know why planning did extra work or skipped expected work. | Generate a plan-phase hook map from the capability registry. |
| Skill surface is large enough to require a generated atlas. | 70 skills/routers in `skills/` and 70 command files in `commands/gsd/`. | New users cannot infer what to use from raw file names alone. | Generate a skill atlas from frontmatter and inventory data. |
| Source/build distinction can confuse contributors. | Authored `src/*.cts` emits generated `gsd-core/bin/lib/*.cjs`; tests ensure outputs. | Editing generated CJS may be lost or drift from source. | Add contributor-facing source-of-truth note near runtime module docs and code review checklist. |
| Config template/schema mismatch can mislead users. | `gsd-core/templates/config.json` includes `gates`, `safety`, and object-shaped `parallelization` sections that are not accepted as central schema leaves by `config-set`. | Users may edit apparently valid template sections that are ignored or warned on. | Reconcile template with schema or explicitly mark template-only/deprecated sections. |
| `config-get` does not mean full effective config. | `config-get` traverses raw files and has a small private defaults map, while `loadConfig()` merges manifests, workstreams, and capability defaults. | Workflow authors and users can get different answers depending on read path. | Document read-path semantics and prefer an explicit effective-config command. |

## Medium Priority

| Gap | Evidence | Why It Matters | Suggested Improvement |
|---|---|---|---|
| Runtime-specific behavior changes invocation and config semantics. | Runtime capabilities and `docs/reference/skill-mapping-matrix.md` show flat/nested skills, hooks/no hooks, config formats, model resolution differences. | A command may behave differently across Claude, Codex, Cursor, Kimi, etc. | Add runtime caveat boxes to command/config docs. |
| Config defaults are not always aligned with valid keys. | Some valid keys have no visible central default, while capability defaults live in capability manifests. | Users cannot distinguish "unset", "false", "defaulted by capability", and "runtime-derived". | Generate default provenance for every key. |
| Capability status has multiple axes. | Installed, surfaced, enabled, active, and gated are separate in capability state/resolution. | Users may disable a capability surface but not understand hook gate state, or vice versa. | Add a capability state glossary and CLI output legend. |
| Some advanced config keys look user-facing but may be internal. | `executor.stall_*`, `manager.flags.*`, runtime state keys, and dynamic feature keys appear in schema. | Docs may accidentally encourage unsupported knobs. | Mark each config key as public, advanced, internal, runtime-state, or deprecated. |
| Worktree behavior is runtime-sensitive. | Docs note non-Claude runtimes may force `workflow.use_worktrees` false. | Users on non-Claude hosts may be confused by parallel execution docs. | Add runtime-specific execution matrix. |
| `bin/install.js` remains a runtime drift hotspot. | Runtime modules exist, but installer edge still owns prompts, target resolution, migrations, rollback behavior, and several runtime-specific cases. | Descriptor-driven behavior can drift from installer special cases. | Continue extracting runtime-specific logic into descriptors/modules and add parity tests. |
| Third-party capability execution is not sandboxed. | Trust model validates and consents executable surfaces but hooks/MCP/command modules run with normal user/runtime permissions after activation. | Users may overestimate safety from validation and integrity checks. | Keep this warning prominent in install UX and handbook docs. |

## Lower Priority

| Gap | Evidence | Why It Matters | Suggested Improvement |
|---|---|---|---|
| Broad docs can lag generated manifests. | Inventory docs say filesystem/manifests are authoritative. | Reader trust degrades if docs and source disagree. | Add docs drift tests for the most important reader-facing counts and config keys. |
| Some command names differ by runtime syntax. | Commands use `gsd:name` while skills use `gsd-name`; namespace routers use bare names. | Users may copy the wrong invocation style. | Add runtime invocation examples per command family. |
| Hook support varies by runtime. | Runtime capabilities show some hosts emit hook events, some have inline/profile-marker-only surfaces. | Capability behavior may be silently unavailable on some hosts. | Add hook support table to capability docs. |
| Generated docs are spread across docs and code. | Capability matrix is generated; inventory is generated/checked; config manifest is generated/shared. | Contributors may not know what to edit versus regenerate. | Add "generated artifact lifecycle" section to contributor docs. |
| Capability source kind `registry` is conceptual, not implemented. | Third-party install supports local/git/npm/tarball in inspected paths; registry is discussed as future/community behavior. | Readers may assume a central registry exists. | Mark registry support as TBD wherever capability install sources are documented. |
| Potential overlay toggle path mismatch needs verification. | Capability state can include overlays; an inspected writer path may validate against the frozen generated registry. | Third-party capability toggles might fail through some writer paths. | Add a targeted test for toggling an installed overlay capability. |

## Questions To Resolve

- Should capability-owned config keys be promoted into the central schema docs,
  or remain separate but generated into one combined reference?
- Which config keys should be hidden from normal users?
- Should `gsd config list` show ownership and source of default?
- Should `gsd surface` explain the difference between surfaced and active?
- Should workflow docs render active capability hooks at runtime for the current
  project config?
