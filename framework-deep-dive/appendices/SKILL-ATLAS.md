# Skill Atlas

This atlas summarizes the shipped skill surface so a reader can understand the
framework without opening every `SKILL.md`. Skill descriptions were cross-checked
against `skills/*/SKILL.md`, `commands/gsd/*.md`, and `docs/INVENTORY.md`.

Current count: 70 generated skills, made of 64 concrete skills plus 6 namespace
routers. The `skills/gsd-*` and `commands/gsd/*.md` stems have parity in this
checkout, but command files carry richer dependency metadata through `requires:`.

## Namespace Routers

These route broad user intent to concrete skills on runtimes that benefit from
smaller top-level skill listings.

| Skill | Purpose | Routes Toward |
|---|---|---|
| `gsd-ns-workflow` | Phase-loop router. | Discuss, plan, execute, verify, phase CRUD, progress, next. |
| `gsd-ns-project` | Project lifecycle router. | New milestone, audit milestone, audit UAT, milestone summary. |
| `gsd-ns-review` | Quality-gate router. | Code review, debug, audit, security, eval, UI review. |
| `gsd-ns-context` | Codebase intelligence router. | Map-codebase, graphify, docs, learnings, MemPalace. |
| `gsd-ns-manage` | Management router. | Config, workspace, workstreams, thread, update, ship, inbox. |
| `gsd-ns-ideate` | Exploration and capture router. | Explore, sketch, spike, spec, capture. |

## Core Workflow Skills

| Skill | What It Does | When To Use | Main Artifacts / Effects |
|---|---|---|---|
| `gsd-new-project` | Initializes a new project with deep context gathering and `PROJECT.md`. | Starting GSD on a new or newly formalized project. | Creates `.planning/`, `PROJECT.md`, `REQUIREMENTS.md`, `ROADMAP.md`, `STATE.md`, `config.json`. |
| `gsd-discuss-phase` | Gathers phase context through adaptive questioning before planning. | Before planning a roadmap phase. | Writes or updates phase `CONTEXT.md`. |
| `gsd-spec-phase` | Clarifies what a phase delivers with ambiguity scoring. | When the phase goal is underspecified before discussion. | Produces `SPEC.md` before discuss/plan. |
| `gsd-mvp-phase` | Frames a phase as a vertical MVP slice using user story and SPIDR splitting. | When the safest plan is a user-visible slice rather than horizontal layers. | Routes into plan-phase with MVP framing; may produce skeleton artifacts. |
| `gsd-ui-phase` | Generates a UI design contract. | For frontend or visual phases. | Produces `UI-SPEC.md`; gated by `workflow.ui_phase`. |
| `gsd-ai-integration-phase` | Generates an AI system design contract. | For phases that build AI/LLM features. | Produces `AI-SPEC.md`; gated by `workflow.ai_integration_phase`. |
| `gsd-sketch` | Sketches UI/design ideas with throwaway HTML mockups. | When visual direction should be explored before formal planning. | Throwaway mockups and optional packaged sketch findings. |
| `gsd-spike` | Spikes an idea through focused experiential exploration. | When feasibility or technical risk needs a cheap experiment. | Throwaway experiments and optional packaged spike findings. |
| `gsd-plan-phase` | Creates executable `PLAN.md` files with research and verification loop. | After phase context exists. | Produces `RESEARCH.md` and `PLAN.md`; uses many plan-time capability hooks. |
| `gsd-plan-review-convergence` | Replans until cross-AI review concerns converge. | When external AI reviewers should drive plan revision cycles. | Reads `REVIEWS.md`; gated by `workflow.plan_review_convergence`. |
| `gsd-ultraplan-phase` | Offloads planning to Claude Code ultraplan cloud, then imports result. | Claude-only beta path for remote planning. | Produces/imports plan artifacts. |
| `gsd-execute-phase` | Executes all plans in a phase with wave-based parallelization. | After plans pass review/checking. | Produces commits, `SUMMARY.md`, build/test evidence. |
| `gsd-verify-work` | Validates built features through conversational UAT. | After execution or when user wants goal-backward validation. | Produces or updates `UAT.md`, `VERIFICATION.md`, issue diagnosis. |
| `gsd-ship` | Creates PR/review/ship preparation after verification passes. | When phase or milestone work is ready to merge. | PR branch/body, review, ship gates, archive/transition effects. |
| `gsd-autonomous` | Runs remaining phases with discuss -> plan -> execute chaining. | For unattended milestone progress. | Drives the core loop according to config gates. |
| `gsd-quick` | Executes quick tasks with GSD guarantees but fewer optional agents. | Small tasks that still need state/commit discipline. | Atomic commits and state tracking, lighter than full phase loop. |
| `gsd-fast` | Executes trivial tasks inline without planning overhead. | Very small edits where subagents/planning would be wasteful. | Direct implementation, minimal GSD ceremony. |

## Phase And Milestone Management

| Skill | What It Does | When To Use | Main Artifacts / Effects |
|---|---|---|---|
| `gsd-phase` | Adds, inserts, edits, or removes phases in `ROADMAP.md`. | Roadmap maintenance. | Updates roadmap and phase directories. |
| `gsd-add-tests` | Generates tests for a completed phase based on UAT criteria. | When implementation exists but test coverage is missing. | Adds test tasks/files and evidence. |
| `gsd-validate-phase` | Retroactively audits and fills Nyquist validation gaps. | When a completed phase needs stronger test/evidence mapping. | Validation artifacts; gated by Nyquist capability. |
| `gsd-secure-phase` | Retroactively verifies threat mitigations. | For completed phases with security concerns. | Security verification artifacts; reads `workflow.security_*`. |
| `gsd-audit-milestone` | Audits milestone completion against original intent. | Before closing a milestone. | Milestone audit report. |
| `gsd-audit-uat` | Audits outstanding UAT and verification items across phases. | Before ship/close, or to find unresolved user-facing checks. | Cross-phase outstanding-items report. |
| `gsd-audit-fix` | Finds, classifies, fixes, tests, and commits audit issues. | When audit results should be turned directly into fixes. | Fix commits and audit-fix report. |
| `gsd-complete-milestone` | Archives a completed milestone and prepares the next version. | After milestone acceptance. | Updates milestone/archive/project state. |
| `gsd-new-milestone` | Starts a new milestone cycle. | When beginning a new version or goal set. | Updates `PROJECT.md`, `STATE.md`, and requirements/roadmap context. |
| `gsd-milestone-summary` | Generates a comprehensive summary from milestone artifacts. | For team onboarding, review, or handoff. | Milestone summary document. |
| `gsd-cleanup` | Archives accumulated phase directories from completed milestones. | After milestone completion or planning directory bloat. | Moves/archives old phase artifacts. |
| `gsd-manager` | Provides an interactive command center for phases. | When managing multiple phases from one terminal. | Dashboard, inline discuss/plan/execute routing. |
| `gsd-workstreams` | Manages parallel workstreams. | When separate lines of work need isolated planning state. | `.planning/workstreams/<name>/` and active pointer changes. |
| `gsd-undo` | Safely reverts phase or plan commits using manifests. | When rolling back GSD-managed work. | Revert commits with dependency checks. |

## Session, Navigation, And Capture

| Skill | What It Does | When To Use | Main Artifacts / Effects |
|---|---|---|---|
| `gsd-next` | Detects project state and routes to the next action. | When unsure what to run next. | Reads state/roadmap/verification/git status; dispatch guidance. |
| `gsd-progress` | Shows progress, advances workflow, or dispatches freeform intent. | For situational status and next-step routing. | Progress summary and optional command routing. |
| `gsd-capture` | Captures ideas, tasks, notes, backlog items, and seeds. | When an idea appears mid-work. | Todo, note, backlog, or seed artifacts. |
| `gsd-stats` | Displays project statistics. | When reviewing progress or project metrics. | Phase/plan/requirement/git/timeline stats. |
| `gsd-pause-work` | Creates context handoff when pausing mid-phase. | Before stopping a session. | `HANDOFF.json`, `.continue-here.md`, optional session report. |
| `gsd-resume-work` | Restores context from previous session. | When continuing paused work. | Reads handoff/state/artifacts and reorients the session. |
| `gsd-thread` | Manages persistent context threads. | For cross-session threads that are not full phases. | Thread state under planning artifacts. |
| `gsd-explore` | Socratic ideation and idea routing. | Before committing exploratory ideas to a roadmap. | May produce captured ideas or route to spike/sketch/spec. |
| `gsd-review-backlog` | Reviews and promotes backlog items. | When deciding what enters the active milestone. | Roadmap/backlog updates. |

## Codebase Intelligence And Memory

| Skill | What It Does | When To Use | Main Artifacts / Effects |
|---|---|---|---|
| `gsd-map-codebase` | Maps codebase structure with mapper agents. | Onboarding a repo or refreshing structure before planning. | `.planning/codebase/*` documents. |
| `gsd-graphify` | Builds, queries, and inspects project knowledge graph. | When relationships across code/artifacts matter. | `.planning/graphs/*`; gated by graphify capability. |
| `gsd-extract-learnings` | Extracts decisions, lessons, patterns, and surprises. | After phases or milestones. | `LEARNINGS.md` and memory-like artifacts. |
| `gsd-mempalace-recall` | Recalls prior decisions/patterns from MemPalace. | Before discuss/plan when cross-session memory is enabled. | `MEMORY-RECALL.md`; gated by `mempalace.enabled`. |
| `gsd-mempalace-capture` | Files phase artifacts into MemPalace and mirrors decision facts. | At phase boundaries when MemPalace is enabled. | MemPalace entries and temporal KG facts. |
| `gsd-docs-update` | Generates or updates documentation verified against codebase. | When project docs need refresh. | Updated docs plus verification report. |
| `gsd-ingest-docs` | Bootstraps or merges planning setup from ADRs, PRDs, SPECs, and docs. | Onboarding a repo with existing planning docs. | Classified/synthesized planning context and conflicts report. |
| `gsd-profile-user` | Generates a developer behavioral profile and discoverable preference artifacts. | When GSD should adapt to a user's working style. | `USER-PROFILE.md`, dev preferences, and generated Claude-profile sections. |

## Quality, Review, Debug, And Recovery

| Skill | What It Does | When To Use | Main Artifacts / Effects |
|---|---|---|---|
| `gsd-review` | Requests cross-AI peer review of phase plans. | When a plan needs external model critique. | `REVIEWS.md`; uses `review.*` config. |
| `gsd-code-review` | Reviews changed source files for bugs, security, and quality. | After execution or before ship. | `REVIEW.md`; `--fix` routes to fixer; gated by `workflow.code_review`. |
| `gsd-debug` | Runs systematic debugging with persistent state. | For defects needing scientific investigation. | Debug session artifacts. |
| `gsd-forensics` | Investigates failed GSD workflows. | When a workflow breaks or produces confusing state. | Forensics report over git/artifacts/state. |
| `gsd-health` | Diagnoses planning directory health and optionally repairs issues. | When `.planning/` may be inconsistent. | Health report and optional repairs. |
| `gsd-import` | Ingests external plans with conflict detection. | When incorporating plans from outside GSD. | Imported/validated plan artifacts or conflict report. |
| `gsd-inbox` | Triages open GitHub issues and PRs against templates. | For project intake/review. | Issue/PR triage output. |
| `gsd-eval-review` | Audits an AI phase's evaluation coverage. | After AI integration work. | `EVAL-REVIEW.md`. |
| `gsd-ui-review` | Retroactively audits implemented frontend code. | After UI implementation. | `UI-REVIEW.md`; gated by `workflow.ui_review`. |

## Configuration, Runtime, And Utility Skills

| Skill | What It Does | When To Use | Main Artifacts / Effects |
|---|---|---|---|
| `gsd-config` | Configures workflow toggles, advanced knobs, integrations, and model profile. | Main configuration entry point. | Mutates `.planning/config.json`. |
| `gsd-settings` | Configures workflow toggles and model profile. | Simpler settings path. | Mutates `.planning/config.json`. |
| `gsd-surface` | Toggles which skills are surfaced. | To reduce or expand runtime skill footprint. | Runtime surface state/profile changes. |
| `gsd-update` | Updates GSD and can sync/reapply local runtime artifacts. | After package upgrades. | Runtime files, changelog, optional reapply/sync effects. |
| `gsd-workspace` | Creates, lists, or removes isolated workspace environments. | For sandboxed or parallel repo workspaces. | Workspace directories and worktree/clone setup. |
| `gsd-pr-branch` | Creates a clean PR branch by filtering `.planning/` commits. | Before opening a source-only PR. | Clean branch for review. |
| `gsd-help` | Shows command reference and usage guide. | Discovery and support. | Help output. |

## Reading Notes

- The skill surface and command surface mostly mirror each other, but exact
  runtime layout depends on installer conversion.
- Namespace routers are not equivalent to concrete skills; they exist to reduce
  eager skill listing cost on specific runtimes.
- Some skills are only useful when the owning capability is enabled or surfaced.
- Several "skills" are management or routing surfaces rather than phase-loop
  steps.
- Fourteen runtimes carry `gsd-` skills; Windsurf emits workflow artifacts
  instead. Flat skill layouts: Claude, Codex, OpenCode, Kilo, Cursor, Copilot,
  Antigravity, CodeBuddy, Kimi. Nested skill layouts: Cline, Qwen, Hermes,
  Augment, Trae.
- Namespace command names are not always identical to generated skill names:
  for example `/gsd-workflow` maps to `gsd-ns-workflow`, and `/gsd-quality`
  maps to `gsd-ns-review`.
