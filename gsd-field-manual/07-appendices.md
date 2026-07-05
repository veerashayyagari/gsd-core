# Appendices

*Reference matter — consulted, not read straight through. A glossary, a one-line digest of every architecture
decision on record, and a map of where each part of the framework lives.*

---

## Appendix A — Glossary & seam index

Each term links to the part of the book where it's explained. Terms marked *(seam)* are the deterministic code
"modules" GSD documents in its own root `CONTEXT.md`.

| Term | One-line meaning | Where |
|---|---|---|
| **Context rot** | The quiet quality decay as an AI's context window fills with its own accumulated output. | I |
| **Fresh context** | The core bet: run heavy work in disposable sub-agents that each start with a clean window. | I |
| **Thin orchestrator** | The coordinating session does no heavy lifting, so its own context grows slowly. | I, II |
| **The phase loop** | Discuss → Plan → Execute → Verify → Ship, repeated one bounded phase at a time. | I, III-A |
| **Milestone / phase** | A milestone is a version's scope; a phase is the next bounded, verifiable chunk of it. | I |
| **The six layers** | User → Commands → Workflows → Agents → CLI tools/`src` → `.planning/` file state. | II |
| **Markdown orchestrates, code computes** | Judgment lives in prose workflows; anything that must be exact lives in tested code. | II, III-A |
| **Workflow** | A long prose "program" the model interprets — the real orchestration logic. | II, III-A |
| **Command ↔ skill mirror** | 70 commands ≈ 70 skills; the skill is the neutral source each runtime converts. | II, III-B |
| **Loop point / lifecycle hook** | One of the twelve `pre`/`post` attachment sites in the loop where features plug in. | II, III-B |
| **The artifact chain** | CONTEXT → PLAN → SUMMARY → UAT: each step's file is the next step's input; the real control flow. | III-A |
| **Wave** | A batch of independent plans executed in parallel; waves run sequentially, plans within one run concurrently. | III-A |
| **STATE.md** | The project's small, living memory file, mutated only through one pure transition function. | III-A |
| **`.planning/`** | The folder of plain-text files that *is* the project's durable state; survives `/clear`. | I, III-A |
| **Absent = enabled** | A missing config flag defaults to *on*; you disable defaults, you don't enable them. | II, III-A |
| **Model profile** | quality / balanced / budget / adaptive / inherit — maps each agent role to a model tier. | III-A |
| **Execution mode** | Interactive (stops at gates) vs autonomous (`--auto`, reasons past them); same workflows. | I, III-A |
| **`WAITING.json`** | A beacon file an autonomous run drops when it's blocked and needs a human. | III-A |
| **Capability (feature)** | An optional loop plugin: config/skills/agents/steps/gates/contributions at loop points. | III-B |
| **Capability (runtime)** | A host descriptor — how GSD installs into Claude, Cursor, Codex, etc. | III-B |
| **Overlay / first-party-wins** | Installed capabilities compose on a frozen base and may only add, never override. | III-B |
| **Trust model** | Consent + integrity + reversibility instead of a sandbox (there is none). | III-B |
| **Hook (host)** | An imperative guard script the *editor* runs on its own tool/session events. | III-B |
| **Degradation model** | Six interface points reduced to full / degraded / unsupported per host; fails closed. | III-B |
| **Interface point** | command · dispatch · model · hooks · state · artifact — the axes a host is graded on. | III-B |
| **MCP server** | A universal driver exposing command + state I/O so any MCP host runs GSD with no plugin. | III-B |
| **must-haves** | A plan's truths, artifacts, and key-links the verifier checks; may add but never subtract from the goal. | III-C |
| **Goal-backward verification** | Work backward from the outcome: what must be TRUE, EXIST, and be WIRED — check against real code. | III-C |
| **Verifier reach = spec reach** | A verifier can only check written-down assertions; raise reliability by widening the spec. | III-C |
| **Probe (edge / prohibition)** | Spec-phase mechanisms that surface omitted edges and unwritten must-NOTs before code exists. | III-C |
| **Source-grounding / drift** | Checking a plan's cited symbols against the live code to catch hallucinations. | III-C |
| **Package-legitimacy gate** | Slopsquatting defense that halts before installing a hallucinated/suspicious package. | III-B, III-C |
| **Worktree / workspace / workstream** | Three isolations: per-executor (ephemeral) · whole-environment (durable) · planning-only (same tree). | III-C |
| **exit-42 / `baseRef: head`** | The worktree base-mismatch halt on a diverged branch, and its one-line fix. | III-C |
| **Research waterfall** | Provider chain by question kind, content-addressed cache, confidence-tiered TTL. | III-C |
| **Intel / graphify** | An opt-in queryable codebase index; an opt-in confidence-tiered knowledge graph. | III-C |
| **Learnings / graduation** | Per-phase lessons that get promoted into project canon when they recur. | III-C |
| **MemPalace** | A temporal knowledge-graph capability — facts carry validity windows; history is preserved. | III-C |
| **Build-at-publish** | `src/*.cts` compiled to gitignored `*.cjs` at publish; source is the truth. | II, III-C |
| **Generated-artifact drift guard** | Committed generated files (registry, contract, skills…) verified against their source by `--check`. | III-B, III-C |
| **Root `CONTEXT.md`** *(seam)* | The framework's own machine-greppable memory ledger — it dogfoods file-based state. | III-C |

---

## Appendix B — ADR digest

Every architecture decision on record, one line each, grouped by theme. Status in parentheses. The full text of
each lives under `docs/adr/`.

**Seams & single-source modules** *(the "one module owns this" pattern)*
- **0001** — Dispatch policy module: single seam for query-execution outcomes *(Accepted)*
- **0002** — Command Contract Validation module *(Accepted)*
- **0003** — Model Catalog module: source of truth for agent profiles & tier defaults *(Accepted)*
- **0004** — Planning Workspace module: single seam for worktree & workstream state *(Accepted)*
- **0006** — Planning Path Projection module *(Accepted)*
- **0008** — Installer Migration module: install-time upgrade safety *(Accepted)*
- **0009** — Shell Command Projection module: runtime-aware OS command rendering *(Accepted)*
- **0010** — File Operation Engine module *(Superseded by 0009)*
- **0012** — CommandRoutingHub: single dispatch seam for CJS command families *(Superseded by 0174)*
- **58** — Runtime Install Policy module: typed install-plan projection *(Accepted)*
- **0656** — Research module: L2-hybrid seam for cached, curated-first research *(Accepted)*
- **766** — Claude Code Plugin Manifest module *(Accepted)*
- **1372** — `markdown-sectionizer` seam: canonical markdown parsing *(Accepted)*
- **1508** — Runtime Artifact Conversion module: per-runtime rewriting *(Accepted)*
- **3660** — Runtime Artifact Layout module: per-runtime placement *(Accepted)*

**The capability system**
- **0010/0011** — Skill Surface Budget module: install-time listing curation & profile staging *(Accepted)*
- **857** — Capability system: five-step loop as core, features as plug-ins at loop points *(Proposed)*
- **894** — Capability declaration format + registry generation *(Proposed)*
- **959** — Capability Command Contribution: a capability's own command family *(Proposed)*
- **1016** — Runtime Capability Descriptor *(Accepted)*
- **1143** — Claude orchestration capability: Workflow tool as a runtime-gated backend *(Proposed)*
- **1213** — Capability State Writer: the write side of capabilities *(Proposed)*
- **1235** — Migrate agent conversion to the descriptor-driven install path *(Accepted)*
- **1244** — Capability Ecosystem: third-party authoring, versioned manifests, URL import/upgrade/remove *(Proposed)*
- **1866** — `agent_skills` dual injection: orchestrator-side + agent self-load *(Accepted)*

**The single-runtime collapse (SDK retirement)**
- **0005 / 0007 / 3524** — SDK seam map / package seam / CJS↔SDK hard seam *(Superseded by 0174)*
- **0174** — Retire the `@opengsd/gsd-sdk` package boundary — single-runtime collapse *(Accepted)*
- **457** — Generation model for `bin/lib/*.cjs` (build-at-publish type safety) *(Accepted)*
- **1239** — GSD as an embeddable orchestration engine *(Accepted)*

**State**
- **1769** — STATE.md Transition module: intent-based transitions over scattered read-modify-writes *(Accepted)*
- **1817** — STATE.md rebuild: the derivability contract (capstone transition) *(Accepted)*

**Verification & probes**
- **22** — Plan-vs-codebase drift guard: defaults & symbol-resolver seam *(Proposed)*
- **550** — Spec-phase probe pattern & prohibition contract *(Accepted)*
- **1606** — Prohibition-enforcement verify-time seam *(Proposed)*

**Testing, quality & portability**
- **452** — Adopt standard ESLint flat-config lint harness *(Accepted)*
- **456** — Test-rigor architecture: deterministic scheduling, antagonistic tier, typed-surface mandate, delete-bad-tests *(Accepted)*
- **1610** — Workflow & agent size-budget ratchet (byte baselines + tier caps) *(Proposed)*
- **1703** — Cross-platform portability enforcement as AST ESLint rules *(Accepted)*

**Security, trust & provenance**
- **227** — Input validation must check semantic shape, not just type *(Accepted)*
- **1411** — Resolution must report provenance, not fall open silently *(Accepted)*
- **1577** — Untrusted-input boundary + opt-in injection blocking *(Proposed)*

**Models, review & routing**
- **443** — Unified cross-provider effort controls & fast-mode-aware routing *(Proposed)*
- **0011** — `review.default_reviewers` scopes the no-flag `/gsd:review` fan-out *(Proposed; PRD Draft)*
- **1517** — Reviewer instances: bounded config for same-adapter multi-model review *(Accepted)*
- **1593** — Skill mapping & converter methodology across runtimes *(Accepted)*
- **1787** — `/gsd:next` smart-entry delegates advancement to `/gsd:progress --next` *(Accepted)*
- **1671** — Dynamic context-management platform *(Proposed)*
- **15** — Cross-AI plan convergence via existing orchestration commands *(Proposed)*

**Release & process**
- **218 / 0175** — Harden release-workflow version validation *(Accepted)*
- **230** — Introduce `next` as a long-lived integration branch *(Proposed)*
- **415** — Prevent stale-base reintroduction of retired runtime tokens *(Accepted)*
- **660** — Release from the head of `next`; immutable tags; `@next` as the RC surface *(Proposed)*

---

## Appendix C — File map

Where each part of the framework lives, for the day you go from reading to changing.

| You're looking for… | It lives in… |
|---|---|
| The commands you type (`/gsd:*`) | `commands/gsd/*.md` — thin front doors |
| Their portable twins | `skills/gsd-*/SKILL.md` — the neutral source runtimes convert |
| The real orchestration logic | `gsd-core/workflows/*.md` — the prose "programs" |
| The specialist sub-agents | `agents/gsd-*.md` — the 34 workers |
| Knowledge injected into agents | `gsd-core/references/*.md` · artifact scaffolds in `gsd-core/templates/` |
| The deterministic domain logic | `src/*.cts` — ~145 TypeScript modules (the "seams") |
| The compiled runtime (not in git) | `gsd-core/bin/lib/*.cjs` — emitted from `src/` at build |
| The CLI everything shells out to | `gsd-core/bin/gsd-tools.cjs` (+ the `gsd_run` shim) |
| The installer & the MCP server | `bin/install.js` · `bin/gsd-mcp-server.js` |
| Feature plugins & host adapters | `capabilities/<id>/capability.json` (`role: feature` or `runtime`) |
| The imperative host guards | `hooks/*.{js,sh}` + `hooks/hooks.json` wiring |
| Generators & invariant linters | `scripts/gen-*.cjs` · `scripts/lint-*.cjs` · `eslint-rules/` |
| The framework's own design record | `docs/adr/` · `docs/design/` · `docs/prd/` |
| User-facing docs (Diátaxis) | `docs/tutorials|how-to|reference|explanation/` |
| The framework's own memory ledger | `CONTEXT.md` (repo root) |
| A *project's* live state | `.planning/` — PROJECT · REQUIREMENTS · ROADMAP · STATE.md · config.json · phases/ |

---

*End of the field manual. You arrived at the surface, went to the bottom, and came back up able to speak to how
GSD is structured, how it feels, how it works — and where it could be better. That last one is why you came.*
