# Configuration Catalog

This catalog explains configuration from implementation-discovered sources. It
combines the central schema manifest, defaults manifest, dynamic key patterns,
and capability-owned keys.

## How Configuration Works

GSD configuration is loaded from `.planning/config.json`, but that file is only
one layer. The effective config comes from:

```text
central defaults
  -> root .planning/config.json
  -> active workstream config overlay
  -> legacy key normalization
  -> capability-owned federated config defaults
  -> runtime/capability surface state for hook activation
```

Important source files:

- `gsd-core/bin/shared/config-schema.manifest.json`: central valid keys and
  dynamic key patterns.
- `gsd-core/bin/shared/config-defaults.manifest.json`: central defaults.
- `src/configuration.cts`: manifest loading and legacy normalization.
- `src/config-loader.cts`: effective config loading and merging.
- `src/config-schema.cts`: central/dynamic/capability key validation.
- `capabilities/*/capability.json`: capability-owned config keys and hook
  `when` gates.

The current central schema manifest contains 100 valid keys. That is not the
whole universe: capability-owned keys federate into the validator when the
capability registry is loaded.

Mutation and reading are not identical paths. `config-set` validates through
`isValidConfigKey(kp, cwd)`, applies type/enum/range guards, blocks dangerous
path segments, and writes dot-paths into `.planning/config.json`. `config-get`
does raw file traversal and has only a small private default map for a few keys,
so workflows often pass explicit shell defaults when they need stable fallback
behavior.

## Central Config Keys

### Core Project And Planning

| Key | Default | What It Entails |
|---|---:|---|
| `mode` | `interactive` | Controls interaction posture. `interactive` asks for confirmations; `yolo` paths can auto-approve more decisions. |
| `granularity` | none central top-level default in manifest, but planning default is `standard` | User-facing phase sizing knob. Legacy `depth` normalizes into this. |
| `planning.granularity` | `standard` | Canonical nested planning granularity default. |
| `parallelization` | `true` | Legacy/global parallelization toggle consumed by older surfaces. More detailed parallelization may be represented elsewhere. |
| `commit_docs` | `true` | Legacy/top-level planning-doc commit toggle. Canonical nested key is `planning.commit_docs`. |
| `planning.commit_docs` | `true` | Whether GSD should commit planning docs as part of workflow operations. |
| `search_gitignored` | `false` | Legacy/top-level alias controlling whether ignored files are searched. |
| `planning.search_gitignored` | `false` | Canonical nested form for searching ignored files. |
| `planning.sub_repos` | `[]` | List of sub-repositories detected or configured for multi-repo planning. |
| `context` | none | Freeform project-specific context injected into agents/workflows. |
| `project_code` | `null` | Short project code used in phase directory naming. |
| `phase_id_convention` | `null` | Phase numbering convention; can enable milestone-prefixed IDs. |
| `phase_naming` | `sequential` | Controls phase directory naming style. |
| `response_language` | none | Preferred response language propagated to agents. |
| `context_window` | `200000` | Assumed model context size; larger values can enable richer context behavior. |
| `claude_md_path` | `./.claude/CLAUDE.md` | Output path for generated Claude memory/instructions. |
| `claude_md_assembly.mode` | none | Controls generated `CLAUDE.md` assembly mode, such as embed versus link. |

### Workflow Toggles

| Key | Default | What It Entails |
|---|---:|---|
| `workflow.plan_check` | `true` | Enables plan verification loop before execution. |
| `workflow.verifier` | `true` | Enables post-execution verifier behavior. |
| `workflow.auto_advance` | `false` | Allows chained phase movement without manual stops in supported workflows. |
| `workflow.node_repair` | `true` | Enables autonomous repair when task verification fails. |
| `workflow.node_repair_budget` | `2` | Maximum repair attempts per failed task. |
| `workflow.human_verify_mode` | `end-of-phase` | Controls whether human verification is mid-flight or consolidated at phase end. |
| `workflow.text_mode` | `false` | Uses plain text numbered prompts instead of richer TUI prompts. |
| `workflow.research_before_questions` | `false` | Runs research before discussion questions rather than after. |
| `workflow.discuss_mode` | `discuss` | Chooses discuss behavior, such as normal discussion versus assumptions mode. |
| `workflow.skip_discuss` | `false` | Lets autonomous flows bypass discuss and create minimal context. |
| `workflow.max_discuss_passes` | `3` | Caps discussion rounds. |
| `workflow.auto_prune_state` | `false` | Automatically prunes stale `STATE.md` entries at phase boundaries. |
| `workflow.use_worktrees` | no default in central manifest output, documented as runtime-sensitive | Enables worktree isolation where supported. Non-Claude runtimes may force or default this off. |
| `workflow.worktree_skip_hooks` | no default in central defaults | Allows executor worktree commits to skip hooks and validate after merge. |
| `workflow.plan_bounce` | `false` | Runs external validation over generated plans. |
| `workflow.plan_bounce_script` | `null` | Script invoked when plan bounce is enabled. |
| `workflow.plan_bounce_passes` | `2` | Number of bounce validation passes. |
| `workflow.plan_chunked` | no central default shown, documented as `false` | Splits planning into short per-plan tasks for crash/hang resilience. |
| `workflow.plan_review_convergence` | no central default shown, documented as `false` | Enables cross-AI plan review/replan convergence command. |
| `workflow.cross_ai_execution` | no central default shown, documented as `false` | Delegates execution to an external AI CLI. |
| `workflow.cross_ai_command` | none | External command template for cross-AI execution. |
| `workflow.cross_ai_timeout` | no central default shown, documented as `300` seconds | Timeout for cross-AI execution. |
| `workflow.subagent_timeout` | `300000` | Timeout in ms for parallel subagent tasks. |
| `workflow.test_command` | none | Explicit project test command for execution/verification gates. |
| `workflow.build_command` | none | Explicit project build command for execution gates. |
| `workflow.mvp_mode` | no central default shown, documented as `false` | Makes phases default to vertical MVP framing. |
| `workflow.context_guard_mode` | `warn` | Controls context pressure guard: warn, auto-pause, or off. |
| `workflow.inline_plan_threshold` | no central default shown | Threshold for inlining small plans versus separate `PLAN.md`. |
| `workflow.context_coverage_gate` | `true` | Enables decision/context coverage checks. |

### Model, Effort, And Routing

| Key | Default | What It Entails |
|---|---:|---|
| `model_profile` | `balanced` | Global model profile: quality, balanced, budget, adaptive, inherit, etc. |
| `runtime` | none | Active runtime for runtime-aware model/profile behavior. |
| `resolve_model_ids` | `false` | Whether abstract model tiers resolve to concrete model IDs. |
| `effort.default` | `high` | Default reasoning effort level. |
| `fast_mode.enabled` | `false` | Enables fast-mode routing. |
| `plan_review.source_grounding` | `true` | Requires source-grounded plan review behavior. |
| `plan_review.source_grounding_authority` | `grep` | Preferred authority for source grounding. |
| `model_policy.provider` | none | Declares known provider or generic/custom policy. |
| `model_policy.budget` | none | Provider-specific budget tier. |
| `model_policy.high` | none | High tier model for generic/custom policy. |
| `model_policy.medium` | none | Medium tier model for generic/custom policy. |
| `model_policy.low` | none | Low tier model for generic/custom policy. |

Dynamic model/routing families:

| Pattern | What It Entails |
|---|---|
| `model_profile_overrides.<runtime>.<opus|sonnet|haiku>` | Runtime-specific model tier override. |
| `models.<planning|discuss|research|execution|verification|completion>` | Per-phase-type tier override. |
| `granularities.<planning|discuss|research|execution|verification|completion>` | Per-phase-type granularity override. |
| `dynamic_routing.enabled` | Enables failure-tier escalation routing. |
| `dynamic_routing.escalate_on_failure` | Allows or disables escalation after soft failure. |
| `dynamic_routing.max_escalations` | Caps escalation attempts. |
| `dynamic_routing.tier_models.<light|standard|heavy>` | Maps dynamic routing tiers to model tier names. |
| `dynamic_routing.<enabled|escalate_on_failure|max_escalations|tier_models.<light|standard|heavy>>` | Exact schema family for all dynamic routing keys. |
| `model_overrides.<agent-id>` | Per-agent model override. |
| `effort.routing_tier_defaults.<light|standard|heavy>` | Reasoning effort default per routing tier. Defaults: light `low`, standard `high`, heavy `xhigh`. |
| `effort.agent_overrides.<agent-id>` | Per-agent effort override. |
| `fast_mode.routing_tier_defaults.<light|standard|heavy>` | Per-tier fast-mode behavior. Defaults: light `true`, standard `false`, heavy `false`. |
| `fast_mode.agent_overrides.<agent-id>` | Per-agent fast-mode override. |
| `model_policy.runtime_tiers.<runtime>.<opus|sonnet|haiku>` | Runtime-specific concrete model and optional effort under model policy. |

### Search And Research Providers

| Key | Default | What It Entails |
|---|---:|---|
| `brave_search` | `false` | Enables or stores Brave Search API config depending on value shape. Runtime may auto-detect keys during project config creation. |
| `firecrawl` | `false` | Enables or stores Firecrawl API config. |
| `exa_search` | `false` | Enables or stores Exa search API config. |
| `tavily_search` | no central default shown | Enables or stores Tavily search API config. |
| `ref_search` | no central default shown | Enables or stores Ref search API config. |
| `perplexity` | no central default shown | Enables or stores Perplexity config. |
| `jina` | no central default shown | Enables or stores Jina fallback/scrape config. |

### Review And Quality

| Key | Default | What It Entails |
|---|---:|---|
| `workflow.code_review_command` | `null` | External code review command for ship/review integration. |
| `code_quality.fallow.enabled` | no central default shown | Enables Fallow code-quality integration. |
| `code_quality.fallow.scope` | no central default shown | Fallow scope, such as phase or project. |
| `code_quality.fallow.profile` | no central default shown | Fallow profile. |
| `code_quality.fallow.mcp` | no central default shown | Fallow MCP integration toggle. |
| `review.ollama_host` | none | Host URL for Ollama review integration. |
| `review.lm_studio_host` | none | Host URL for LM Studio review integration. |
| `review.llama_cpp_host` | none | Host URL for llama.cpp review integration. |
| `review.default_reviewers` | none | Default reviewer set for review commands. |
| `review.max_prompt_tokens` | none | Global prompt budget for review. |
| `review.max_prompt_tokens_per_reviewer` | none | Shared per-reviewer budget setting. |

Dynamic review families:

| Pattern | What It Entails |
|---|---|
| `review.models.<cli-name>` | Model id to inject for a reviewer CLI. |
| `review.max_prompt_tokens_per_reviewer.<reviewer-slug>` | Per-reviewer prompt budget. |
| `review.reviewer_instances.<instance-name>.<cli|model|agent>` | Named reviewer instance configuration. |
| `review.reviewer_instances.<instance-name>.<cli|model|agent> (#1517)` | Exact schema family label for named reviewer instance configuration. |

### Git, Workspaces, Hooks, And Status

| Key | Default | What It Entails |
|---|---:|---|
| `git.branching_strategy` | `none` | Branching model for phase/milestone work. |
| `git.base_branch` | `null` | Base branch used for branch/worktree operations. |
| `git.create_tag` | `true` | Whether milestone completion creates tags. |
| `git.phase_branch_template` | `gsd/phase-{phase}-{slug}` | Template for phase branches. |
| `git.milestone_branch_template` | `gsd/{milestone}-{slug}` | Template for milestone branches. |
| `git.quick_branch_template` | `null` | Template for quick-task branches. |
| `hooks.context_warnings` | `true` | Enables context warning hooks. |
| `hooks.workflow_guard` | `false` | Enables workflow guard hooks. |
| `statusline.show_last_command` | none | Displays last command in statusline when supported. |
| `statusline.context_position` | none central default in valid list; docs default often `end` | Controls where context status appears. |
| `manager.flags.discuss` | none | Manager default flag for discuss behavior. |
| `manager.flags.plan` | none | Manager default flag for plan behavior. |
| `manager.flags.execute` | none | Manager default flag for execute behavior. |
| `executor.stall_detect_interval_minutes` | none | Interval for executor stall detection. |
| `executor.stall_threshold_minutes` | none | Threshold before executor is considered stalled. |
| `ship.pr_body_sections` | `[]` | Additional PR body sections for ship output. |

### Features, Agent Skills, Capabilities, And Security

| Key | Default | What It Entails |
|---|---:|---|
| `features.thinking_partner` | none | Enables conditional thinking-partner behavior. |
| `features.global_learnings` | none | Enables global learnings injection behavior. |
| `learnings.max_inject` | none | Caps injected learnings. |
| `agent_skills_security.trusted_global_roots` | none | Trusted roots for global agent skill injection. |
| `capabilities.strict_known_registries` | `null` | Policy for permitted third-party capability sources; `null` permissive, `[]` lockdown. |
| `capabilities.auto_update` | `false` | Enables automatic capability update behavior. |
| `security.injection_blocking` | `false` | Top-level security object for read-injection scanner blocking behavior. Distinct from workflow security keys. |
| `graphify.build_timeout` | none | Timeout for graphify build operations. |
| `graphify.auto_update` | `false` | Enables automatic graph updates where graphify hooks are active. |

Dynamic feature/agent families:

| Pattern | What It Entails |
|---|---|
| `agent_skills.<agent-type>` | Injects project/global skills into a GSD agent type. |
| `features.<feature_name>` | Feature-specific toggles not individually enumerated centrally. |
| `claude_md_assembly.blocks.<section>` | Per-section assembly override for generated Claude memory. |

## Capability-Owned Config Keys

Capability keys are declared in `capabilities/*/capability.json`. Some are also
present in the central schema during migration; others exist only through
federated capability config.

| Capability | Keys | What They Entail |
|---|---|---|
| `ai-integration` | `workflow.ai_integration_phase` | Enables AI-SPEC design contract workflow. |
| `assumption-delta` | `workflow.assumption_delta` | Enables advisory checkpoint for singular/plural, required/optional, or derived/chosen modeling shifts. |
| `code-review` | `workflow.code_review`, `workflow.code_review_depth` | Enables code review command and selects quick/standard/deep depth. |
| `drift` | `workflow.drift_threshold`, `workflow.drift_action`, `workflow.schema_drift_gate`, `workflow.plan_drift_precheck` | Controls stale structure/schema drift checks at plan and execute hook points. |
| `external-job` | `external_job.enabled`, `external_job.backend`, `external_job.artifact_dir`, `external_job.submit_timeout_ms`, `external_job.poll_timeout_ms` | Enables async external-job producer and scheduler backend/timeouts. |
| `gap-analysis` | `workflow.post_planning_gaps` | Enables non-blocking post-planning coverage report. |
| `graphify` | `graphify.enabled` | Enables graphify capability surfaces and graph operations. |
| `intel` | `intel.enabled` | Enables code-intelligence store and plan-time intel hook. |
| `mempalace` | `mempalace.enabled`, `mempalace.memory_mode`, `mempalace.wing`, `mempalace.recall_on_discuss`, `mempalace.recall_on_plan`, `mempalace.capture_artifacts`, `mempalace.mirror_kg`, `mempalace.cross_project_tunnels`, `mempalace.diary_journal`, `mempalace.auto_capture_hooks` | Controls cross-session/cross-project memory recall, capture, temporal KG mirroring, diary, and future passive hooks. |
| `nyquist` | `workflow.nyquist_validation` | Enables validation coverage audit after verification. |
| `pattern-mapper` | `workflow.pattern_mapper` | Enables pre-planning codebase pattern mapper agent. |
| `profile-pipeline` | `profile-pipeline.enabled` | Enables developer behavioral profiling pipeline. |
| `research` | `workflow.research` | Enables optional phase research before planning. |
| `schema-gate` | `workflow.schema_push_detection` | Enables schema-relevant plan injection to prevent DB/schema drift. |
| `security` | `workflow.security_enforcement`, `workflow.security_asvs_level`, `workflow.security_block_on` | Enables threat mitigation verification, ASVS level, and blocking threshold. |
| `tdd` | `workflow.tdd_mode` | Enables TDD planning heuristics and execution gate enforcement. |
| `ui` | `workflow.ui_phase`, `workflow.ui_review`, `workflow.ui_safety_gate` | Enables UI design contract, UI review, and blocking UI safety gates. |

## Runtime State Keys

| Key | What It Entails |
|---|---|
| `workflow._auto_chain_active` | Runtime state marker for auto-chain behavior. It appears in schema manifests for completeness but should be treated as internal runtime state, not normal user config. |

## Important Gaps

- Central schema and defaults are separate, so a valid key may have no obvious
  default in `config-defaults.manifest.json`.
- Some keys documented in `docs/CONFIGURATION.md` are capability-owned rather
  than central.
- Some capability-owned keys are highly behaviorally important but invisible if
  a reader only inspects the central schema.
- The central schema does not currently attach ownership metadata, so ownership
  must be inferred from capability manifests, consumer modules, and docs.
- Runtime-specific defaults, especially worktree and model behavior, can differ
  from the neutral config template.
- `gsd-core/templates/config.json` contains object-shaped `gates`, `safety`,
  and `parallelization` sections that are not central schema leaves. If treated
  as ordinary config keys, `gates` and `safety` are likely ignored by
  `loadConfig()` and may warn as unknown top-level keys.
- `docs/CONFIGURATION.md` documents some non-schema/template-only keys such as
  `context_profile`, `gates.*`, `safety.*`, and `parallelization.*` subfields.
  `config-set` does not accept those nested keys through the central validator.
- `cmdConfigNewProject()` has a hardcoded materialization layer and API-key
  detection; it is not simply a clone of `config-defaults.manifest.json`.
