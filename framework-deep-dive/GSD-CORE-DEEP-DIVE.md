# GSD Core Deep Dive

## How To Read This

This handbook is written from the outside in. It starts with what a user sees,
then follows the main journey, then opens the framework and explains how the
pieces are built. You should not need to keep other files open to understand the
system at a high level. The appendices in this folder exist for complete
catalogs and trace evidence.

The most important idea is that GSD Core is not just a CLI and not just a prompt
pack. It is a framework made of prompt artifacts, skills, commands, workflows,
agents, CLI tools, planning files, runtime adapters, hooks, capabilities,
configuration, and generated install artifacts.

## Part 1: The Surface

### What GSD Core Is

GSD Core is a context-engineering and spec-driven development framework for AI
coding agents. Its job is to keep long-running software work coherent by forcing
work through a disciplined loop:

1. Discuss what the phase should mean.
2. Plan how it should be built.
3. Execute the plan in scoped work units.
4. Verify the result against the original goal.
5. Ship, archive, and move to the next phase.

The framework's central bet is that AI coding work degrades when one chat
session accumulates too much unrelated context. GSD Core responds by keeping
the main session lean, writing durable state to `.planning/`, and delegating
research, planning, execution, verification, review, and audits to specialized
agents with fresh context windows.

The user-facing experience is a set of commands or skills such as
`gsd-new-project`, `gsd-discuss-phase`, `gsd-plan-phase`,
`gsd-execute-phase`, `gsd-verify-work`, and `gsd-ship`. Underneath that surface,
those commands load workflow files, spawn agents, call `gsd-tools`, and mutate
planning artifacts.

### The User's Mental Model

A user should think in terms of milestones, phases, plans, and evidence.

- A project starts with persistent planning memory: `PROJECT.md`,
  `REQUIREMENTS.md`, `ROADMAP.md`, `STATE.md`, and `config.json`.
- A milestone is a coherent unit of project progress.
- A phase is a roadmap item inside that milestone.
- A phase gets discussed, planned, executed, verified, and shipped.
- Every step produces artifacts that can survive a context reset.
- The current position is recoverable from `.planning/STATE.md` and neighboring
  phase files.

GSD Core wants the AI to be less improvisational and more traceable. It turns
"please build this" into a documented chain:

```text
intent -> requirements -> roadmap -> context -> plan -> summary -> verification -> ship
```

That chain is why the framework has so many files. The files are not incidental;
they are the memory substrate.

### Skill Atlas At A Glance

The repository ships 70 skill surfaces: 64 concrete skills plus 6 namespace
routers. The routers help runtimes with limited skill routing pick a category
first:

| Router | What It Covers |
|---|---|
| `gsd-ns-workflow` | Phase loop: discuss, plan, execute, verify, phase, progress. |
| `gsd-ns-project` | Project lifecycle: milestones, audits, summaries. |
| `gsd-ns-review` | Quality gates: code review, debug, audit, security, eval, UI. |
| `gsd-ns-context` | Codebase intelligence: map, graphify, docs, learnings, memory. |
| `gsd-ns-manage` | Configuration, workspaces, workstreams, thread, update, ship, inbox. |
| `gsd-ns-ideate` | Exploration and capture: explore, sketch, spike, spec, capture. |

The concrete skills fall into the same operating categories:

- Core workflow: new project, discuss, plan, execute, verify, ship, quick, fast,
  autonomous.
- Phase and milestone management: phase CRUD, add tests, validate, secure,
  audit, complete milestone, new milestone, cleanup, workstreams, undo.
- Session and navigation: next, progress, capture, stats, pause, resume, thread,
  explore, backlog.
- Codebase intelligence: map-codebase, graphify, extract-learnings, MemPalace,
  docs update, docs ingest.
- Quality and recovery: review, code-review, debug, forensics, health, import,
  inbox, eval review, UI review.
- Configuration and runtime management: config, settings, surface, update,
  workspace, pr-branch.

The full skill atlas is in `appendices/SKILL-ATLAS.md`, but the key idea is
simple: skills are the runtime-facing entry points. They usually correspond to
files in `commands/gsd/`, which load workflow files from `gsd-core/workflows/`.

## Part 2: The Main Journey

### The Full Phase Loop

The canonical journey is:

```text
gsd-new-project
  -> gsd-discuss-phase
  -> gsd-plan-phase
  -> gsd-execute-phase
  -> gsd-verify-work
  -> gsd-ship
```

`gsd-new-project` initializes the durable project memory. It asks questions,
may run research, writes requirements and roadmap artifacts, and sets the
starting state.

`gsd-discuss-phase` turns a roadmap phase into concrete implementation context.
Depending on configuration, it can ask adaptive questions, run assumptions mode,
use plain text rather than UI prompts, or be skipped in autonomous mode.

`gsd-plan-phase` is where the framework becomes strict. It researches if
enabled, may invoke UI, TDD, security, pattern-mapping, gap-analysis, drift, or
other capability hooks, spawns the planner, writes `PLAN.md`, and runs a plan
checker unless verification is disabled.

`gsd-execute-phase` runs the plan. It can execute in waves, use worktrees on
Claude-compatible runtimes, call build/test gates, repair failed tasks, and
produce `SUMMARY.md` evidence.

`gsd-verify-work` checks whether the implementation actually satisfies the
phase goal. It leans on UAT, verifier behavior, Nyquist validation, security
checks, UI review, and human verification policy.

`gsd-ship` prepares the result for merge. It can run review, enforce security
or ship gates, create a PR branch, and archive or summarize work.

### What Gets Created

The user-facing phase loop creates and consumes a planning filesystem. The
important artifacts are:

| Artifact | Role |
|---|---|
| `.planning/PROJECT.md` | Project identity, goals, constraints, and long-lived context. |
| `.planning/REQUIREMENTS.md` | Requirements, acceptance criteria, and durable intent. |
| `.planning/ROADMAP.md` | Milestone and phase decomposition. |
| `.planning/STATE.md` | Current position, status, decisions, blockers, metrics, and continuity. |
| `.planning/config.json` | Project behavior knobs for models, workflow gates, runtime behavior, review, git, hooks, and capabilities. |
| `.planning/phases/<N>/CONTEXT.md` | Phase-specific decisions and implementation context. |
| `.planning/phases/<N>/RESEARCH.md` | Research output, when research is enabled or explicitly requested. |
| `.planning/phases/<N>/PLAN.md` | Executable plan tasks, gates, risks, verification criteria. |
| `.planning/phases/<N>/SUMMARY.md` | Execution summary and evidence from implementation. |
| `.planning/phases/<N>/UAT.md` | Human/user acceptance evidence. |
| `.planning/phases/<N>/VERIFICATION.md` | Goal-backward verification and unresolved issues. |
| `.planning/codebase/*` | Codebase maps, structure, patterns, and generated intelligence. |
| `.planning/graphs/*` | Graphify knowledge graph artifacts. |
| `.planning/intel/*` | Structured codebase intel when the intel capability is enabled. |
| `.planning/async-jobs/*` | External job manifests when the external-job capability is enabled. |

The design choice is deliberately file-based. There is no database and no
server dependency for core state. This makes state inspectable by humans,
available to agents, and commit-friendly.

### Where Configuration Starts To Matter

Configuration changes the loop at nearly every stage:

- `workflow.research` decides whether planning gets a research step.
- `workflow.plan_check` decides whether plans are verified before execution.
- `workflow.verifier` and `workflow.human_verify_mode` affect post-execution
  verification.
- `workflow.use_worktrees` affects parallel execution isolation.
- `workflow.node_repair` and `workflow.node_repair_budget` affect automatic
  repair behavior.
- `workflow.ui_phase`, `workflow.ui_review`, and `workflow.ui_safety_gate`
  activate UI-specific contracts and review.
- `workflow.tdd_mode` activates TDD planning and execution gates.
- `workflow.security_enforcement`, `workflow.security_asvs_level`, and
  `workflow.security_block_on` affect threat-model verification.
- `model_profile`, `models.*`, `model_policy.*`, `dynamic_routing.*`,
  `model_overrides.*`, and effort/fast-mode keys affect model selection and
  reasoning behavior.
- Capability-owned keys such as `mempalace.enabled`, `external_job.enabled`,
  `intel.enabled`, and `graphify.enabled` activate extension systems that are
  not all visible in the central config manifest.

The important critique point is that configuration cannot be understood from
`docs/CONFIGURATION.md` alone. The canonical central schema lives in
`gsd-core/bin/shared/config-schema.manifest.json`; defaults live in
`gsd-core/bin/shared/config-defaults.manifest.json`; capability-owned keys live
in `capabilities/*/capability.json` and federate into the loader at runtime.

## Part 3: Under The Hood

### Command And Skill Routing

The user does not invoke TypeScript modules directly. The surface starts as
skills or commands:

```text
skills/<name>/SKILL.md
commands/gsd/<name>.md
```

Those files include frontmatter, a description, allowed tools, arguments, and
prompt instructions. The body usually points at a workflow:

```text
gsd-core/workflows/<workflow>.md
```

Different runtimes expose the same conceptual command differently. Claude Code
uses custom slash commands and skills. Codex consumes skills. Other runtimes
may receive slash commands, workflow files, nested skills, flat skills, or
runtime-specific command wrappers. Runtime conversion is handled by installer
and runtime artifact modules rather than by each workflow.

At the CLI layer, `gsd-tools` dispatches command families through routers and
the command routing hub. `src/command-routing-hub.cts` is intentionally a
no-throw, pure-result dispatch boundary. It routes CJS handlers, returns typed
result variants, and centralizes error kinds such as `UnknownCommand`,
`InvalidArgs`, `HandlerRefusal`, and `HandlerFailure`.

### Workflow Orchestration

Workflow files are the orchestration layer. They are not supposed to do heavy
implementation work. They load context, call `gsd-tools`, spawn the right
agents, collect outputs, and update state.

This thin-orchestrator design matters because prompt files are loaded into
model context. Large eager prompts increase attention cost and quality risk.
The architecture docs explicitly treat workflow byte budgets as quality
protection, not just token-billing protection.

The pattern looks like this:

```text
skill/command
  -> workflow markdown
    -> gsd-tools init/query/update
    -> agent spawn
    -> write or verify .planning artifacts
    -> transition state
```

### Agents And References

Agents are specialized prompt definitions in `agents/gsd-*.md`. They include
role, tool permissions, and expectations for output. The repository currently
ships 34 agent definitions.

References in `gsd-core/references/` are shared knowledge chunks used by
workflows and agents. Examples include gate definitions, context budgeting,
model profile resolution, verification patterns, TDD guidance, git integration,
untrusted input boundaries, and planning templates.

The agent model is the framework's answer to context rot. Instead of making one
chat session read everything, the orchestrator asks a focused agent to read a
bounded set of artifacts and produce a bounded output.

### CLI Tools Layer

`gsd-core/bin/gsd-tools.cjs` is the runtime CLI surface that workflows and
agents call. The CLI centralizes operations that prompt files should not
reimplement:

- State parsing and mutation.
- Phase lookup and lifecycle.
- Roadmap parsing and updates.
- Config loading and setting.
- Model resolution.
- Capability state.
- Worktree and workspace helpers.
- Validation, verification, audit, graphify, intel, research, and other domain
  commands.

The authored source for many of these modules is in `src/*.cts`. The build
compiles those files to `gsd-core/bin/lib/*.cjs`, which is what the package
ships and what runtime workflows call.

### Configuration System Deep Dive

Configuration has three overlapping layers:

1. Central valid keys.
2. Dynamic key patterns.
3. Capability-owned federated keys.

Central valid keys are defined in
`gsd-core/bin/shared/config-schema.manifest.json`. The current manifest lists
100 valid central paths. Defaults are defined in
`gsd-core/bin/shared/config-defaults.manifest.json`.

Dynamic patterns allow controlled families of keys, such as:

- `agent_skills.<agent-type>`
- `review.models.<cli-name>`
- `models.<planning|discuss|research|execution|verification|completion>`
- `granularities.<planning|discuss|research|execution|verification|completion>`
- `dynamic_routing.*`
- `model_overrides.<agent-id>`
- `review.reviewer_instances.<instance>.<cli|model|agent>`
- `model_policy.runtime_tiers.<runtime>.<opus|sonnet|haiku>`

Capability-owned keys are declared in `capabilities/*/capability.json`.
Examples include:

- `external_job.enabled`
- `external_job.backend`
- `mempalace.enabled`
- `mempalace.memory_mode`
- `intel.enabled`
- `graphify.enabled`
- `workflow.assumption_delta`
- `workflow.schema_push_detection`
- `workflow.schema_drift_gate`
- `workflow.plan_drift_precheck`

`src/config-loader.cts` merges defaults, project config, workstream overlays,
legacy-key normalization, and federated capability config. It also protects
against prototype-pollution keys such as `__proto__`, `constructor`, and
`prototype`. Its provenance-aware path can report whether config came from a
workstream, root project config, built-in defaults, or global defaults.

`src/config-schema.cts` is a thin adapter over the manifest and capability
registry. `isValidConfigKey` returns true if the key is central, runtime state,
dynamic, or capability-owned.

There is one subtle split: `config-set` validates through the schema path,
while `config-get` performs raw file traversal with only a small private
defaults map for a few keys. Effective config and a single `config-get` output
are therefore not always the same concept unless the workflow supplies explicit
fallbacks.

The most important behavior is precedence:

```text
built-in defaults
  -> root .planning/config.json
  -> active workstream config overlay
  -> explicit runtime/config commands
  -> capability gates and surface state for hook activation
```

The full catalog is in `appendices/CONFIGURATION-CATALOG.md`. For critique,
the key issue is discoverability. A reader must inspect manifests, config
loader code, capability manifests, and docs to get the complete picture.

### Capability System

Capabilities are the framework's extension and ownership mechanism. A
capability declares:

- an id, title, role, tier, version, and engine compatibility;
- skills and agents it owns;
- config keys it owns;
- loop steps, contributions, or gates;
- runtime compatibility and install behavior.

There are two broad capability roles:

- Feature capabilities extend the loop. Examples: UI, TDD, security, research,
  pattern mapper, gap analysis, MemPalace, external job, intel, graphify.
- Runtime capabilities adapt GSD to host tools. Examples: Claude, Codex,
  Cursor, Copilot, Kimi, Kilo, OpenCode, Windsurf, Antigravity, Qwen, Cline,
  Augment, Trae, Hermes, CodeBuddy.

The committed first-party registry is generated into
`gsd-core/bin/lib/capability-registry.cjs`. Installed third-party capabilities
are overlaid at runtime by `src/capability-loader.cts`. First-party capability
ids and owned surfaces win over overlays. Third-party capabilities must pass
validation, engine compatibility, integrity, and consent checks. If a skipped
capability declared a gate, the loader records a fail-closed gate so the loop
does not silently proceed as if the gate passed.

Capabilities interact with config in two ways:

1. They declare config keys.
2. Their hooks use `when` clauses that read those config keys.

That is why the configuration system and capability system must be documented
together.

Third-party capability installation is declaration-first and non-executing
during validation. Sources can be local, git, npm, or tarball; the conceptual
`registry` source kind is not implemented in this checkout. Executable surfaces
such as hooks, MCP servers, and command modules are disclosed and consented,
but they are not sandboxed after activation. They run with the normal
permissions of the user and host runtime.

### Build, Install, And Runtime Artifacts

The repository has an authored source tree and a runtime/package artifact tree.

Authored TypeScript modules live in `src/*.cts`. The build uses
`tsconfig.build.json`, with:

```json
{
  "rootDir": "src",
  "outDir": "gsd-core/bin/lib",
  "module": "nodenext",
  "target": "ES2022"
}
```

The `.cts` extension lets TypeScript emit `.cjs` runtime modules. This is part
of the ADR-457 "generated CJS single source" architecture: migrate behavior out
of hand-written CJS and into typed source while preserving the runtime CJS
surface.

`package.json` exposes:

- `gsd-core` -> `bin/install.js`
- `gsd-tools` -> `gsd-core/bin/gsd-tools.cjs`
- `gsd_run` -> `gsd-core/bin/gsd_run`
- `gsd-mcp-server` -> `bin/gsd-mcp-server.js`

The installer emits different artifact layouts for different runtimes. Some
runtimes get flat skills, some nested skills, some slash commands, some
workflow files, some hooks config, and some only profile markers. The runtime
mapping is documented in `docs/reference/skill-mapping-matrix.md` and encoded
in runtime artifact/layout/conversion modules.

The current mapping is not uniform: Windsurf emits workflows rather than
skills; Claude, Codex, OpenCode, Kilo, Cursor, Copilot, Antigravity,
CodeBuddy, and Kimi use flat skill layouts; Cline, Qwen, Hermes, Augment, and
Trae use nested skill layouts. Hook support also varies by runtime.

This split is essential to understand when debugging:

```text
src/*.cts                 authored source
  -> npm run build:lib
gsd-core/bin/lib/*.cjs    generated runtime modules
  -> installer
runtime config dirs       emitted skills, commands, hooks, settings
```

### Quality, Tests, And Safety

GSD Core has a broad test suite under `tests/`, with unit, integration,
install, security, and slow suites selected by `scripts/run-tests.cjs`.

The test runner has important build behavior: before tests run, it ensures
`src/*.cts` has emitted corresponding `gsd-core/bin/lib/*.cjs` files and
ensures hook distribution artifacts exist. This closes clean-checkout and
deleted-output failure modes.

Safety appears in several layers:

- Config validation and unknown-key warnings.
- Runtime capability validation and consent.
- Untrusted input isolation references and tests.
- Security scanning and prompt-injection tests.
- Plan checking before execution.
- Verification and UAT after execution.
- Worktree safety and base-branch checks.
- ESLint rules for portability, test rigor, raw filesystem removal, ad hoc
  markdown parsing, hardcoded temp paths, and fragile shell usage.

The framework is designed as defense in depth: prompt instructions, CLI
validators, config gates, capability gates, tests, and lint rules all share
responsibility.

## Part 4: Critique

### System Strengths

GSD Core has several strong architectural choices:

- It makes context explicit through artifacts rather than relying on chat
  memory.
- It separates user surface, orchestration, agents, references, CLI tools, and
  runtime adapters.
- It uses generated manifests and tests to reduce drift.
- It treats runtime diversity as an artifact conversion problem rather than
  duplicating logic per host.
- It has a real extension model through capabilities instead of only hardcoded
  flags.
- It has many safety backstops around config, install, execution, and
  verification.

### Main Gap Themes

The main critique themes are:

1. Configuration is too distributed to understand from one place.
2. Some capability-owned keys are behaviorally important but not obvious from
   the central config schema.
3. The skill surface is large enough that users need a generated atlas.
4. Runtime-specific behavior is crucial but easy to miss when reading only the
   core workflow docs.
5. There are two implementation surfaces, authored TS and generated CJS, and
   readers need to know which one to edit.
6. The installer still owns enough runtime-specific edge behavior that it
   remains a likely drift point while descriptor-driven modules continue to
   absorb responsibility.
7. Broad docs can lag generated manifests unless drift checks cover the exact
   reader-facing claim.
8. The capability system is powerful but conceptually dense: install profile,
   surface state, config gates, hook activation, consent, and overlay loading
   are separate axes.

### Improvement Opportunities

The highest-leverage improvements are:

1. Generate a public configuration catalog from central schema plus capability
   manifests.
2. Generate a skill atlas from command and skill frontmatter.
3. Add ownership metadata to config schema entries.
4. Mark every config key as user-facing, advanced, runtime-state, capability
   owned, or legacy.
5. Add a doc drift test that compares `docs/CONFIGURATION.md` against central
   and capability-owned config keys.
6. Add diagrams for the core phase loop, config loading, capability activation,
   and runtime install artifact generation.
7. Make the source/build distinction prominent in contributor docs.

The rest of this folder turns those critique themes into lookup tables and
trace evidence.
