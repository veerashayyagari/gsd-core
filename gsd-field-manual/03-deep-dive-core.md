# Part III-A — The Core, and How It Varies

*In which we open the box in the middle of the map: the loop itself, how it bends between modes, how it
remembers, and every dial you can turn.*

---

This is the engine room. In Part II you saw the six-layer map from altitude; now we descend into the three
middle layers — workflows, agents, and the file state they read and write — and take the loop apart to see what
actually makes it turn. By the end of this chapter you'll understand how a phase advances, why the *same*
workflow behaves so differently in interactive versus autonomous mode, how `STATE.md` survives a crash, and
what every knob in `config.json` does to the flow. It's the longest chapter in the book, because it's the most
important one. Take it in one sitting if you can; it builds.

---

## 1 · The loop engine — a program written in prose

Here is the fact that reorganizes everything: **there is no loop engine.** No function named `runPhaseLoop()`,
no scheduler, no state machine ticking somewhere in a process. The loop *is* a set of Markdown files in
`gsd-core/workflows/`, and the "engine" that runs them is the AI model itself, reading a workflow the way you'd
read a recipe and doing what it says.

When you typed `/gsd:execute-phase 1` for Snip, the matching workflow file was loaded, in full, into the model's
context, and the model became its interpreter. That file is a program written in English: *load the phase
context, figure out the plans, group them into waves, spawn an executor for each, collect the results, update
the state, move on.* The prose is the control flow; the model executes it step by step.

This is why Part II insisted the orchestrator is *thin*. A workflow never does the heavy lifting itself. Its
job is to call `gsd-tools init` to load a compact context payload, decide which specialist to spawn, hand that
specialist a focused prompt, collect what it returns, and call `gsd-tools state …` to write down what changed.
The clever, deterministic, error-prone work is pushed *down* to the CLI and *out* to fresh agents. The
orchestrator stays a coordinator, so its own context grows slowly and it never rots.

There's a lovely tell in how the workflows are maintained: each one is held under a strict byte budget — tiers
of roughly 38K, 54K, and 90K bytes. Not to save money on tokens, but to protect the model's *attention.* The
framework applies its own core thesis — context rot — to its own instructions: a leaner, higher-signal workflow
produces a sharper orchestrator. GSD distrusts long context even in its own prompts.

### The contract: five steps, twelve points

Underneath the prose is a small, frozen, machine-readable spine called the **loop-host contract.** Every
workflow file declares, in its frontmatter, which step it belongs to, which extension points it exposes, which
agent roles it uses, and which core artifact it produces and consumes. A build script scrapes those declarations
from all the workflows and freezes them into a single contract. Here is the whole thing:

| Step | Extension points | Agent roles | Consumes → Produces |
|---|---|---|---|
| **discuss** | `discuss:pre` · `discuss:post` | orchestrator | — → **CONTEXT.md** |
| **plan** | `plan:pre` · `plan:post` | researcher, planner, checker | CONTEXT.md → **PLAN.md** |
| **execute** | `execute:pre` · `wave:pre` · `wave:post` · `execute:post` | executor, verifier | PLAN.md → **SUMMARY.md** |
| **verify** | `verify:pre` · `verify:post` | orchestrator | SUMMARY.md → **UAT.md** |
| **ship** | `ship:pre` · `ship:post` | orchestrator | UAT.md → — |

Two plus two plus four plus two plus two is the **twelve loop points** you first heard named in Part II.
*Execute* gets four because it has an inner sub-loop: `wave:pre` and `wave:post` fire once per wave. These
points are attachment sites — the places where optional features and hooks bind to the loop without editing a
line of it. (That's Part III-B's story; file it away.)

### The artifact chain *is* the control flow

Look at that last column and read it top to bottom: **CONTEXT → PLAN → SUMMARY → UAT.** Each step's output file
is the next step's required input. Discuss decides things and writes `CONTEXT.md`; plan reads those decisions
and writes `PLAN.md`; execute reads the plans and writes `SUMMARY.md`; verify reads the summaries and writes
`UAT.md`; ship reads the acceptance result and closes the loop.

This chain is the real control flow of GSD — not because a program enforces it, but because *the loop passes
state between steps through files on disk, never through memory.* And that single design choice is what makes
fresh context possible. Because every hand-off is a file, any step can begin in a completely blank context and
reconstruct everything it needs by reading the previous step's artifact. The conversation is disposable; the
chain of files is the program's memory. Chapter 1's promise — *agents rely on the file, not on memory* — is
this chain, made literal.

### Fanning out: the wave model

The most parallel moment in GSD is execution, and it's worth watching closely. Every `PLAN.md` carries two bits
of frontmatter: a `wave` number and a `dependencies` list. When execute runs, the orchestrator's whole job is:
*discover the plans, read their dependencies, group them into waves, spawn one executor per plan, and reconcile
the results.* The rule is two-dimensional and simple:

- **Within a wave: parallel.** Independent plans run at the same time.
- **Across waves: sequential.** Wave 2 doesn't start until every worktree from Wave 1 is merged and removed.

For Snip's Phase 1, the link-store plan and the create-endpoint plan had no dependency between them, so both ran
in Wave 1, simultaneously, each in its own fresh executor.

Now, the subtle part — what "fresh" concretely means at the moment of spawn. The orchestrator does **not** paste
file *contents* into the executor's prompt. It passes *paths*: "here is your plan file, here is `PROJECT.md`,
here is `STATE.md`, here is the config." The executor opens them itself, spending its *own* clean context budget
to read exactly what it needs. This is why the orchestrator stays lean — it's shuttling filenames, not files —
and why each executor gets a full, uncontaminated window to think in. (On the newest million-token models the
loop notices the larger window and *enriches* the spawn prompt with more inlined context — prior summaries, the
phase research — so executors can be aware of each other's work in ways that wouldn't fit a 200K window. The
loop reads `context_window` from config and switches behavior at the half-million-token mark.)

Two invariants keep parallel work from corrupting itself. First, on Claude Code each executor runs in its own
git *worktree* — a separate working copy on a temporary branch — so simultaneous agents can't tread on each
other's files. Second, and quietly crucial: **executors are forbidden to write `STATE.md` or `ROADMAP.md`.**
Those are shared files; if parallel agents wrote them, the last writer would clobber the rest. So the executors
only ever write their *own* per-plan `SUMMARY.md`, and when the wave finishes the orchestrator becomes the
**single writer** — it merges each worktree back, runs a build-and-test gate over the integrated result, and
*only if the tests pass* records the plans complete and advances the state. A plan is never marked done while
the integration tests are red. Then, and only then, the next wave forks.

That's the engine: prose orchestration, a frozen step contract, a chain of files that carries the state, and a
disciplined fan-out-and-reconcile for parallelism. Everything else in this chapter is either how that engine
*bends,* how it *remembers,* or how you *tune* it.

---

## 2 · Execution modes — the same flow, bent

Here's something that surprises people: interactive Snip and an unattended, hands-free Snip run **the same
workflow files.** There is no separate "autonomous engine." Autonomy is the same loop with its human gates
answered differently. Understanding *where* it bends is understanding the mode system.

The big fork is **interactive versus autonomous.** In interactive mode — the way you built Snip — the loop
stops at its gates and waits for you: it interviews you during Discuss, it shows you the roadmap and waits for
*Approve,* it asks you the verification questions. In autonomous mode (`/gsd:autonomous`, or `--auto` on the
router), it doesn't stop: it reasons out answers and keeps going, chaining phase into phase, finishing the
milestone on its own.

The single most important substitution happens at Discuss. Interactive Discuss *interviews* you. Autonomous
Discuss runs what GSD calls **smart discuss**: instead of asking, it analyzes the phase, generates a *proposed
answer* to each gray area with its rationale, and lays them out as a batch. Left fully to itself, it accepts its
own well-reasoned recommendations — even resolving the ones it's unsure about — and writes an **identical**
`CONTEXT.md` to the one an interview would have produced. Same artifact, a quarter of the interaction. (Discuss
has a third setting too, orthogonal to the mode: **assumptions mode**, where a sub-agent reads a handful of your
actual files, forms confidence-tagged assumptions, and asks you only to correct the wrong ones. Codebase-first
instead of interview-first, same `CONTEXT.md` out.)

But the reassuring half of the story is what does **not** bend. The guards stay up regardless of mode:

- The **plan checker** still runs — autonomous plans are verified before execution, same as interactive ones.
- The **package-legitimacy gate** still stops dead for a suspicious dependency. GSD will not silently install a
  flagged package, no matter how autonomous the run.
- When the loop hits something it genuinely cannot decide, or finds real gaps in the work, it **pauses and
  asks** rather than guessing — offering you *answer / skip this phase / stop.*

When it pauses like that in an unattended run, it can drop a small `WAITING.json` marker on disk — a
machine-readable "I'm blocked, here's the question, here are the options" — that an external watcher can notice,
answer, and clear, so the run continues. That little file (we'll meet it again under State) is how a run hands
off between "running by itself" and "needs a human" without losing its place.

Here is the fork, side by side:

| | **Interactive** | **Autonomous** (`/gsd:autonomous`, `--auto`) |
|---|---|---|
| Who drives | you run each command, `/clear` between phases | the loop chains discuss→plan→execute across all remaining phases |
| Discuss | interview (or assumptions), you pick the gray areas | smart discuss: proposes answers, auto-accepts its recommendations |
| Roadmap / "ready?" gates | shown and awaited | skipped |
| Verification | you answer the checks | a passing phase advances with no prompt |
| Plan checker | on | **still on** |
| Package-legitimacy gate | stops | **still stops** |
| Genuine unknowns / gaps | you decide | **pauses and asks** (answer / skip / stop) |
| Hand-off when blocked | n/a | drops `WAITING.json` for a watcher |
| After the last phase | you run the milestone commands | audit → complete → cleanup run automatically |

Autonomy isn't the only tempo. There are lighter ones — `/gsd:quick` for a small tracked task with real
guarantees but no phase overhead, `/gsd:fast` for a trivial inline edit with no sub-agents at all — and you'll
meet the whole family in the catalog later in this chapter. The point for now is structural: **modes are not
different code paths; they're the same loop with its gates and its Discuss step configured differently.** Which
brings us to the thing that configures them — but first, the loop's memory.

---

## 3 · State & memory — how a phase survives a crash

Everything the loop knows about Snip lives in files, and the beating heart of those files is a single small one:
`.planning/STATE.md`. It's read at the start of every workflow and rewritten after every meaningful action, and
it's deliberately kept under a hundred lines — a digest of where you are, not an archive of everything you've
done. It has two halves: YAML frontmatter that machines parse (the current `status`, the active phase and next
action, a progress block, timestamps) and a Markdown body that humans read (a "Current Position" section with a
little `[████░░░░] 40%` bar, recent decisions, blockers, and where the last session stopped).

### One doorway for every change

Every mutation of `STATE.md` goes through a single **pure function** — think of it as `transition(text, intent)
→ text`. It takes the current file as a string and an *intent* — one of a small set like `beginPhase`,
`advancePlan`, `completePhase`, `sync`, or `rebuild` — and returns the new file as a string. It touches no
disk and takes no lock; it's just text in, text out. `advancePlan`, for instance, finds "Plan 2 of 3,"
increments it, and — when the last plan is done — flips the status to "ready for verification" instead.
`completePhase` closes a phase, advances to the next, and recomputes the progress percentage from the roadmap.

### Preserve versus derive — the one table that governs it all

Here's the design idea worth admiring. When GSD rewrites `STATE.md`, some fields should be *recomputed* from
scratch (the timestamp, the plan counts) and some should be *preserved* (a note a human wrote, a status an
executor set). How does it know which is which? A single frozen table where **every field is one row,**
classifying it two ways: its *source* (invented fresh, read from config, parsed from the body, counted from
disk, or curated by a human) and its *preservation rule* (always recompute, keep-unless-the-source-changed,
never-overwrite-unless-named, or overwrite-only-if-it's-still-a-placeholder).

So `last_updated` is "always recompute." The plan counts are "recompute by counting `PLAN.md` and `SUMMARY.md`
files on disk." The progress block is "preserve always" — it's a *ratchet* that never regresses. And because the
whole policy lives in that one table, **adding a new field to `STATE.md` is one new row** — no new branching,
no scattered special cases. The system even refuses to touch a field that has no row, forcing whoever adds a
field to declare its behavior first. There's a matching instinct in the body text: a value that looks like one
of GSD's own templates ("Ready to execute," a bare date) may be overwritten, but anything else is assumed to be
human- or executor-authored and is left untouched. The framework is careful never to clobber a sentence a
person wrote.

### The lock, and the beacon

Because parallel agents exist, two of them might try to rewrite `STATE.md` at once. GSD guards it with a
lockfile created atomically — the first writer wins the lock, does its entire read-modify-write (including the
disk scan that counts plans), and only then releases. Holding the scan *inside* the lock is deliberate: it
closes a window where a second agent could slip a new file in between the first agent's count and its write,
stamping stale numbers. The lock is even PID-aware: a waiter that finds the lock held checks whether the holder
is actually alive, and makes a careful three-way decision — a live holder is never robbed mid-write; a
crashed-and-dead holder's lock is reclaimed promptly; a half-created lock is given a moment to finish being born
before it's treated as an orphan. A process that exits clears any locks it held, so a crash never strands the
file. It is a surprising amount of care for one text file — and it exists because that one text file is the
project's memory, and a corrupted memory is the one failure the whole design cannot tolerate.

Separate from the lock is a cooperative signal: `WAITING.json`, the beacon from the last section. When the loop
needs a human it writes this little file — `{ status: "waiting", question, options, phase, since }` — and when
the human answers, it's deleted. The lock coordinates *concurrent writes;* the beacon coordinates *human
attention.* They're different mechanisms for different problems, and it's worth keeping them apart.

### The folder, over time

Zoom out and the whole `.planning/` directory is the project's memory, and it fills in a predictable order. The
root holds the project-long files — `PROJECT.md`, `ROADMAP.md`, `REQUIREMENTS.md`, `STATE.md`, `config.json` —
plus transient ones like `HANDOFF.json` (written by pause, consumed once by resume) and `WAITING.json`. Each
phase gets a folder whose contents appear in loop order, tracing the artifact chain exactly:

| Artifact | Written by (step) | Read by |
|---|---|---|
| `CONTEXT.md` | discuss | researcher, planner, checker |
| `RESEARCH.md` | plan | planner, plan-checker |
| `NN-PP-PLAN.md` (one per plan) | plan | executor, checker, verifier |
| `NN-PP-SUMMARY.md` (one per plan) | execute | verifier, progress, later planners |
| `VERIFICATION.md` | verify | gates replanning; humans |
| `UAT.md` | verify | resumed later by audit |

And the payoff, the thing all of this exists for: **everything in `.planning/` survives `/clear`.** Only the
in-context conversation is wiped. After a clear, the next command re-reads `STATE.md`, the config, the phase
artifacts, and any `WAITING.json`, and picks up exactly where it left off. Nothing in the working conversation
is load-bearing. The files are.

---

## 4 · The control panel — `config.json`

Snip's `.planning/config.json` is a small file with an outsized reach. It's the framework's control panel, and
learning to read it is learning to predict what the loop will do. One structural fact makes the whole thing
tractable, so absorb it first: **almost nothing in `config.json` is read by an AI at "decide what to do" time.**
The config is consumed by the *deterministic* orchestrator — the workflow prose and the `gsd-tools` CLI — which
then chooses which agents to spawn, with which model, behind which gate. So every option below doesn't change a
model's *judgment;* it changes the *shape of the spawn graph or the gate graph.* Turn a knob, and an agent
appears or vanishes, a gate opens or closes.

### Absent means enabled

You met this rule in Part II; here's how it actually works. When the loader needs a setting, it reads
`config.get(key) ?? default`, and for the quality-and-safety steps the built-in default is `true`. So *omitting*
a key enables it. The good, careful path is the zero-config path — the guards are all on unless you deliberately
switch them off:

> **On by their absence:** research, plan-checking, verification, test-coverage validation, UI design gates,
> pattern-mapping, auto-repair of failed tasks, decision-coverage checking, code review, security enforcement,
> parallel execution, doc-committing, context warnings, and every confirmation gate.

The mirror image is just as deliberate: the *behavior-changing* features default **off**, so that upgrading GSD
never silently changes how an existing project runs. You opt *into* YOLO mode, auto-advancing, TDD mode, MVP
mode, cross-AI execution, the memory palace, the knowledge graph — none of them turn on by surprise.

### The dials that matter, grouped by what they bend

There are around a hundred options; you don't need them all in your head (the full catalogue lives in the
configuration reference if you ever do). What you need is the *shape* of the panel — which groups exist and how
each bends the flow:

- **Run posture.** `mode` is the master tempo: `interactive` stops at gates, `yolo` auto-answers all of them.
  `granularity` (`coarse`/`standard`/`fine`) tells the roadmapper how finely to slice — coarse is two-to-four
  big phases, fine is six-to-ten small ones. `context_window` at half a million or more unlocks the adaptive
  enrichment you saw in the wave model.

- **The step graph** (`workflow.*`). This is the heart of the panel, because each toggle literally adds or
  removes an agent from the loop. Turn off `research` and no researcher spawns before planning. Turn off
  `plan_check` and plans go to execution *unverified.* Turn off `verifier` and a phase is marked done with no
  post-execution check. Turn off `node_repair` and a task that fails verification is not auto-fixed. Each of
  these is a real subtraction from the defense-in-depth stack — which is exactly why they default on.

- **Mode selectors.** `discuss_mode` picks interview versus assumptions. `tdd_mode` makes the planner tag tasks
  red-green-refactor and the executor enforce that sequence. `mvp_mode` makes each phase a thin vertical slice
  instead of a horizontal layer. `human_verify_mode` decides whether human checks block mid-execution or wait
  for the end. `auto_advance` chains the steps without stopping.

- **Gates** (`gates.*`). This is the granular alternative to the global `yolo` switch. Each named gate —
  confirm the project, confirm the roadmap, confirm each plan, confirm a transition — can be individually turned
  off, removing exactly that one human stop while leaving the others in place.

- **Parallelism & git.** `parallelization` and `use_worktrees` govern the wave machinery (and, notably,
  `use_worktrees` *fails closed* — force it on a non-Claude runtime and execution aborts rather than silently
  degrading, because worktree isolation is a Claude-only primitive). `git.branching_strategy` decides whether
  the loop works on your current branch, cuts a branch per phase, or cuts one per milestone.

- **Review, security, and capabilities.** `code_review` and `cross_ai_execution` bring in a second set of eyes
  (or a second AI entirely). `security_block_on` sets the threat severity that will *halt* a phase.
  Capability master-switches like `mempalace.enabled`, `intel.enabled`, and `graphify.enabled` light up whole
  optional subsystems — all off by default, all inert when off.

A useful wrinkle to notice: not every guard *blocks.* The plan-phase decision-coverage gate genuinely halts the
loop if a decision went unimplemented; but its cousins — the drift pre-check, the assumption-delta nudge, the
verify-side coverage check — are *advisory*: they write a warning and let a green phase stay green. Same family
of settings, opposite blocking semantics. When you're reasoning about whether a config change can stall a run,
that distinction is the one to hold.

### Choosing the brains: model profiles

One corner of the panel deserves its own paragraph, because it's where cost and quality are actually traded:
model selection. Every one of the ~33 agents carries three model tiers in a catalog — call them golden,
balanced, and budget. A single setting, `model_profile`, picks which column the whole roster uses:

- **quality** runs the golden column — the strongest model for every decision agent.
- **balanced** (the default) spends big only where it counts — the planner gets the top model, most agents get
  the mid one, the cheap mappers get the small one.
- **budget** runs lean — mid-tier for writing code, small for research and verification.
- **adaptive** routes each agent by its own weight class.
- **inherit** makes every agent follow the session's model — the escape hatch for non-Anthropic providers.

When an agent spawns, the orchestrator asks the CLI to *resolve* its model, and the answer comes from a
precedence waterfall: a precise per-agent override wins first; then dynamic routing if it's on; then a per-phase
tier; then the global profile; then the runtime's own default. And there's a genuinely clever option layered on
top — **dynamic routing** — which starts each agent at its normal tier but, if the orchestrator judges the
result a *soft* failure (a verification came back inconclusive, a plan-check raised a flag), re-spawns it one
tier *up.* You pay top-model rates only for the hard cases. (Reasoning *effort* rides a completely separate axis
from model tier, incidentally — you can run a small model at maximum effort — but that's a detail for the
curious.)

### How the panel loads, and a warning

Config resolves in layers: an optional per-workstream override sits over the project's `config.json`, which sits
over the built-in defaults; a machine-global defaults file seeds new projects at creation time. Two behaviors
are worth knowing because they surprise people. First, **reading the config can rewrite it** — on load, GSD
quietly migrates legacy key names to their modern form and self-corrects a few settings (if `.planning/` is
gitignored, it forces doc-committing off, whatever the file says). Second, config edits **hot-reload**: a
change to `config.json` mid-session takes effect immediately, via a hook that re-reads the file and tells the
running agent what changed — because forcing a `/clear` to pick up a config edit would destroy the very
continuity the whole framework exists to protect. The working context survives; the configuration updates
beneath it.

That's the control panel. Everything the loop does, it does because some dial — set or absent — told it to.

---

## 5 · The full set of doors — a scenario catalog

You've now seen the engine and its dials. The last two sections are breadth: every front door, and every worker
behind them. You will rarely need to *memorize* the doors — `/gsd:next` reads your state and picks the right one
for you — but knowing which command fits which situation is real fluency, and it's how you drive GSD
deliberately instead of just following its nose.

Mechanically, every command is a thin skill that hands off to a workflow, which shells out to `gsd-tools`, which
dispatches through a single pure routing hub to a deterministic handler. Above the flat list sit two "smart"
doors — `/gsd:next`, a state-aware *launcher* that classifies your situation and dispatches one command, and
`/gsd:progress --next`, the *advancement engine* that actually moves the loop forward — plus six namespace
meta-routers (`ns-workflow`, `ns-project`, and friends) that exist purely to keep the model's menu cheap to
list. Everything below remains directly invocable.

Here's the map of doors, by the situation you're in:

**The core loop.** `spec-phase` (pin down a fuzzy *what* first) · `discuss-phase` (settle the *how*) · `ui-phase`
(a visual design contract for frontend work) · `plan-phase` (turn decisions into checked plans) · `execute-phase`
(build it in waves) · `verify-work` (conversational UAT) · `code-review` (bugs/security/quality on the diff) ·
`ship` (assemble the PR).

**Project & milestone lifecycle.** `new-project` (start from nothing) · `new-milestone` (begin the next version
cycle) · `audit-milestone` (confirm the definition of done) · `complete-milestone` (archive and tag) ·
`milestone-summary` (an onboarding write-up) · `resume-work` / `pause-work` (clean stop and restart).

**Reshaping the roadmap.** `/gsd:phase` is the one door: it appends a phase by default, `--insert` slots an
urgent decimal phase mid-milestone, `--remove` deletes and renumbers, `--edit` changes a field. `/gsd:capture`
is its sibling for ideas — todos, backlog items, notes, and forward-looking "seeds" that resurface when their
trigger arrives.

**Shaped phases.** `mvp-phase` (a thin vertical slice that proves the whole stack) · `ai-integration-phase`
(build an LLM feature — spawns a framework-selector, researchers, and an eval-planner) · `discovery-phase` (a
high-uncertainty opening pass) · `ultraplan-phase` (offload planning to the cloud, in beta).

**Exploring before you commit.** `explore` (Socratic thinking, no plan yet) · `spike` (throwaway experiments to
prove a technical approach is feasible) · `sketch` (throwaway HTML mock-ups to compare design directions). The
last two can *wrap up* their findings into a reusable project-local skill.

**Ad-hoc work.** `fast` (a trivial inline edit, no sub-agents) · `quick` (a small task with real guarantees but
no roadmap entry) · `autonomous` (build the whole remaining roadmap hands-free).

**Recovery & debugging.** `debug` (scientific-method bug hunting with persistent state) · `forensics` (a
post-mortem when a *GSD workflow itself* went wrong) · `undo` (safe rollback of GSD commits) · `health`
(validate or repair a corrupted `.planning/`, or probe context usage).

**Onboarding a brownfield repo.** `map-codebase` (four mappers write a portrait of your stack and conventions) ·
`import` (bring in an external plan file) · `ingest-docs` (bootstrap `.planning/` from scattered ADRs and specs).

**Review & audit.** `review` (cross-AI peer review of a plan) · `plan-review-convergence` (iterate a plan until
reviewers are satisfied) · `secure-phase` (verify threat mitigations landed) · `ui-review` (visual audit of
built frontend) · `eval-review` (audit an AI phase's eval coverage) · `validate-phase` (fill test-coverage
gaps) · `audit-fix` (find issues *and* fix them).

**Docs & knowledge.** `docs-update` (write docs, then verify every claim against the code) · `extract-learnings`
(harvest reusable patterns) · `graphify` / `intel` (build a queryable graph or index of the codebase) ·
`mempalace` (recall prior decisions, or file this phase's artifacts into a temporal memory).

**Navigation & settings.** `next` / `progress` / `manager` (find or advance your position; run several phases
from one terminal) · `settings` / `config` / `surface` (turn features on and off) · `workspace` / `workstreams`
(isolate parallel lines of work) · `update` (upgrade GSD).

That's the whole surface. Seventy-odd doors, but they cluster into maybe a dozen intentions — and now you can
name the intention and find the door.

---

## 6 · The workforce — the agent roster

Behind those doors work thirty-four specialists. You never call them directly; a workflow spawns them, each in a
fresh context, each with only the tools its job requires (a doc-writer that only reads and writes Markdown gets
no shell access — small blast radius by design). They are the "fresh context" of Chapter 1 made into a staff.
Here they are, by trade:

- **Researchers** gather knowledge before decisions: the **project-researcher** (the domain, before a roadmap),
  the **phase-researcher** (how to build one phase), the **advisor-researcher** (one gray-area decision), the
  **domain-** and **ai-researcher** (for AI phases), the **ui-researcher** (a design contract), and the
  **research-synthesizer** that merges their parallel output.

- **Planners & checkers** turn knowledge into a verified plan: the **planner** (writes the executable plans),
  the **roadmapper** (slices the project into phases), the **plan-checker** (proves a plan will hit the goal),
  the **pattern-mapper** (maps new files to existing analogs), the **assumptions-analyzer** (for assumptions-mode
  discuss), and the AI-phase pair, the **framework-selector** and **eval-planner**.

- **The executor** stands alone: it's the agent that actually writes Snip's code, one per plan, committing
  atomically, handling deviations, respecting checkpoints.

- **Verifiers** ask whether the goal was met: the **verifier** (goal-backward, post-execution), the
  **integration-checker** (do the phases connect end-to-end?), the **nyquist-auditor** (is test coverage
  adequate?), the **ui-checker** (does the build match the design contract?).

- **Reviewers & auditors** are the retroactive quality gates: the **code-reviewer**, the **security-auditor**,
  the **ui-auditor**, the **eval-auditor** — each producing a scored report on a different axis.

- **Fixers & debuggers** repair what's broken: the **code-fixer** (applies review findings), the **debugger**
  (scientific-method bug hunting), and its **debug-session-manager** (runs the whole debug loop in isolation so
  your main context stays lean).

- **Doc agents** keep the words honest: the **doc-writer**, the **doc-verifier** (checks every documented claim
  against the live code), and the ingest pair, the **doc-classifier** and **doc-synthesizer**.

- **Mappers** build codebase intelligence: the **codebase-mapper** (the brownfield portrait) and the
  **intel-updater** (the queryable index).

- **And a few specialists**: the **user-profiler** (learns your working style) and the **mempalace-curator**
  (files long-term memory at ship time).

---

You've now been all the way down into the core. You've seen the engine (a program written in prose, driving a
chain of files), how it bends between its modes without changing its code, how it protects its one irreplaceable
memory, every dial that shapes it, all the doors into it, and the whole workforce behind them.

But the core is only half the framework. GSD's other half is what makes it *extensible* and *portable* — how you
bolt new behavior onto that twelve-point loop without editing it, and how this one Claude-shaped codebase runs on
sixteen different tools. That's the next descent. Turn to Part III-B.
