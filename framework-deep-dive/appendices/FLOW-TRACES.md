# Flow Traces

These traces describe the major journeys at a level useful for critique. They
are intentionally implementation-aware but not line-by-line call graphs.

## 1. New Project

```text
skill/command: gsd-new-project
  -> commands/gsd/new-project.md
  -> gsd-core/workflows/new-project.md
  -> project research / questioning / roadmapping agents
  -> gsd-tools state/config/roadmap/template helpers
  -> .planning/PROJECT.md
  -> .planning/REQUIREMENTS.md
  -> .planning/ROADMAP.md
  -> .planning/STATE.md
  -> .planning/config.json
```

Purpose: convert a raw project idea into durable project memory and a roadmap.

Important configuration:

- `granularity` and `planning.granularity` influence phase sizing.
- `model_profile`, `models.*`, and model policy keys influence agent model
  choices.
- Search provider keys can affect research depth.
- `planning.commit_docs` affects whether planning artifacts are committed.

Critique angle: project creation is the reader's first exposure to many
configuration defaults, but many defaults are not visible unless the generated
config is inspected.

## 2. Core Phase Loop

```text
gsd-discuss-phase
  -> CONTEXT.md
gsd-plan-phase
  -> RESEARCH.md
  -> PLAN.md
  -> plan checker result
gsd-execute-phase
  -> implementation commits
  -> SUMMARY.md
  -> build/test gates
gsd-verify-work
  -> UAT.md
  -> VERIFICATION.md
gsd-ship
  -> review/security/ship gates
  -> PR branch or archive artifacts
```

Important configuration:

- `workflow.discuss_mode`
- `workflow.max_discuss_passes`
- `workflow.skip_discuss`
- `workflow.research`
- `workflow.plan_check`
- `workflow.plan_chunked`
- `workflow.verifier`
- `workflow.human_verify_mode`
- `workflow.auto_advance`
- `workflow.context_guard_mode`
- `workflow.build_command`
- `workflow.test_command`

Capability hook points:

- `discuss:pre`, `discuss:post`
- `plan:pre`, `plan:post`
- `execute:wave:post`, `execute:post`
- `verify:post`
- `ship:pre`, `ship:post`

Critique angle: the loop is conceptually clean, but the actual behavior depends
on config plus capability gates. A one-page phase-loop diagram should include
capability hook points, not only the five user-facing verbs.

## 3. Plan Phase With Capabilities

```text
gsd-plan-phase
  -> phase lookup and config load
  -> plan:pre hooks
     -> research
     -> UI phase
     -> pattern mapper
     -> intel
     -> MemPalace recall
     -> assumption delta
     -> schema gate
     -> drift precheck
     -> security/TDD contributions
  -> planner agent
  -> PLAN.md
  -> plan:post hooks
     -> gap analysis
     -> external job planning contribution
     -> MemPalace capture
  -> plan checker unless skipped
```

Important configuration:

- `workflow.research`
- `workflow.ui_phase`
- `workflow.ui_safety_gate`
- `workflow.pattern_mapper`
- `intel.enabled`
- `mempalace.enabled`
- `workflow.assumption_delta`
- `workflow.schema_push_detection`
- `workflow.plan_drift_precheck`
- `workflow.security_enforcement`
- `workflow.tdd_mode`
- `workflow.post_planning_gaps`
- `external_job.enabled`

Critique angle: `plan-phase` is the densest extension point. It should be the
first target for diagrams and generated hook documentation.

## 4. Runtime Install

```text
npx @opengsd/gsd-core@latest
  -> bin/install.js
  -> install engine/profile/runtime modules
  -> command and skill conversion
  -> runtime config dir
  -> emitted commands/skills/hooks/settings
  -> shared gsd-core runtime files
```

Runtime differences:

- Some runtimes receive flat skills.
- Some receive nested namespace router skills.
- Some receive slash-command workflows.
- Some support hooks; some have no lifecycle hook surface.
- Some resolve model IDs at install time rather than invocation time.

Critique angle: runtime behavior is not a peripheral concern. It changes how a
user invokes skills, how hooks work, and when config changes take effect.

## 5. Configuration Load

```text
loadConfig(cwd)
  -> central defaults
  -> root .planning/config.json
  -> optional workstream .planning/workstreams/<name>/config.json
  -> legacy-key normalization
  -> federated capability config defaults
  -> validation / unknown-key warnings
  -> resolved config object
```

Important modules:

- `src/configuration.cts`
- `src/config-loader.cts`
- `src/config-schema.cts`
- `src/federated-config.cts`
- `src/capability-loader.cts`

Critique angle: a config key can be valid because it is central, dynamic,
runtime-state, or capability-owned. Documentation that only lists central keys
will be incomplete.

## 6. Capability Activation

```text
capability.json
  -> generated first-party registry
  -> optional installed overlay registry
  -> install profile says whether capability is installed
  -> runtime surface says whether skill is surfaced
  -> config gate says whether hook is active
  -> loop resolver decides active steps/contributions/gates
```

The key distinction:

- Installed means files exist and the capability is known.
- Surfaced means the runtime exposes its skills/commands.
- Enabled means installed plus surfaced.
- Active means enabled and config gates allow the relevant hook.

Critique angle: capability status has too many overloaded words in normal
conversation. Docs should preserve the precise axes.

## 7. Verification And Ship

```text
execution summary
  -> verifier / UAT / Nyquist / UI / security checks
  -> unresolved issue diagnosis
  -> optional repair or audit-fix
  -> ship review / branch / PR preparation
```

Important configuration:

- `workflow.verifier`
- `workflow.human_verify_mode`
- `workflow.nyquist_validation`
- `workflow.ui_review`
- `workflow.security_enforcement`
- `workflow.security_block_on`
- `workflow.code_review`
- `workflow.code_review_depth`
- `review.*`
- `ship.pr_body_sections`

Critique angle: verification is both a workflow stage and a set of optional
capabilities. The docs should clarify which checks are always part of the loop
and which are capability-gated.
