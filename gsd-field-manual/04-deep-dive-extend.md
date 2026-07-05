# Part III-B — Extending and Porting

*In which the two double-edged words from Part II are finally taken all the way apart: how you bolt new
behavior onto the loop without editing it, and how one Claude-shaped codebase runs on sixteen different tools.*

---

The core you just toured is only half the framework. The other half is what lets GSD *grow* and *travel* —
add a feature to the loop without touching the loop, and run that same loop on a tool it was never written for.
Both halves are built from the two words Part II warned you were overloaded, and this is the chapter where the
overloading finally pays off instead of confusing you. **Capability** turns out to name the whole
extend-and-port system: feature capabilities extend the loop, runtime capabilities port it. **Hook** names two
attachment systems: the loop points that features plug into, and the host guardrails that police the runtime.
We'll take all four apart, in that order.

---

## 1 · Feature capabilities — changing the loop without editing it

Recall the twelve loop points from the last chapter — the `pre`/`post` pairs around Discuss, Plan, Execute,
Verify, Ship, plus the wave pair inside Execute. Those points exist for exactly one purpose: so that optional
behavior can attach to the loop *without a single edit to the loop's own workflow files.* A **feature
capability** is a unit of that optional behavior — a self-contained folder with a `capability.json` manifest
that declares what it adds and where. Test-driven development is one. A memory palace is one. Security auditing
is one. Eighteen ship with GSD, and you (or a third party) can write more.

The design rule is a clean line: something becomes a capability when it can be switched on or off as one piece
and owns its own name, skills, agents, config keys, or commands; it stays in the core when it's the reliability
substrate every project needs. TDD is optional, so it's a capability. Fresh context is not optional, so it's
core.

### The anatomy of a feature

A feature capability can declare up to seven kinds of thing, and each is a way of touching the loop:

| Part | What it does |
|---|---|
| **config** | Config keys the capability *owns* (e.g. `mempalace.enabled`). They merge into `config.json` through the federated overlay; a collision with a core key or another capability fails the build. |
| **skills / agents** | Skill and agent definitions it brings with it. Exactly one capability may own any given name. |
| **steps** | "At this loop point, run this skill / agent / command." A step is **purely additive** — it can add work but can never halt or redirect the loop on its own. |
| **gates** | "At this loop point, check this condition — and maybe *block*." Gates are the only part that can stop the loop. |
| **contributions** | "Inject this text fragment into this agent's prompt at this loop point." Changes what an agent is *told*, not what runs. |
| **hooks** | Host-lifecycle scripts it registers (the guardrail kind — §3). |
| **commands** | A whole `gsd-tools` command family — the one and only place a third-party capability's *own code* executes. |

Watch it work with the memory palace. `mempalace` owns nine `mempalace.*` config keys (a master
`mempalace.enabled`, off by default, plus refinements), two skills (`mempalace-recall`, `mempalace-capture`),
and one agent (the curator). It plugs **steps** in at five points: capture what was decided at `discuss:post`,
recall relevant memories at `plan:pre` (writing a `MEMORY-RECALL.md` the planner then reads), capture again at
`plan:post` and `verify:post`, and run the curator at `ship:post`. It plugs **contributions** in at two more:
a fragment into the orchestrator at `discuss:pre` ("recall what you already know before gathering new
context"), and one into the verifier at `execute:wave:post` ("persist confirmed problem→fix pairs"). Switch
`mempalace.enabled` on and the loop quietly grows all of that behavior. Switch it off and every one of those
attachments vanishes from the active set — the loop runs byte-for-byte as if the capability didn't exist. That
is what "changing the loop without editing it" means, made concrete.

### Additive steps, blocking gates

The steps/gates split is the load-bearing distinction. A **step** can only add; if a capability needs to
*stop* the loop, it must use a **gate.** And gates come in three flavors, with a rule that matters:

- A **query** gate runs deterministic first-party code and may block.
- A **predicate** gate checks a declarative condition (does this artifact exist? does this config key equal
  that? does this file's frontmatter say `threats_open: 0`?) and may block.
- An **agent-verdict** gate asks an LLM to judge — and is *forced advisory,* never allowed to block.

The principle is worth stating plainly, because it recurs throughout GSD: **a non-deterministic check may never
halt the loop.** Only deterministic gates get to say "stop." The `security` capability uses this for real — a
`ship:pre` predicate gate that refuses to ship unless the security report shows zero open threats — while
`tdd`'s review checkpoint is deliberately advisory.

There's a second, orthogonal safety knob on every step and gate: `onError`, which says what happens if the hook
*itself* throws (as opposed to what its verdict is). `onError: skip` means a crashing optional step is simply
dropped and the loop proceeds — so a flaky memory-palace call never wedges your phase. `onError: halt` is for
controls that must not be silently swallowed: the security capability halts if its *check* errors, because a
security gate that fails to run is not the same as one that passed.

### How they compose: first-party always wins

Now the interesting part — what happens when capabilities stack, including ones a third party wrote and you
installed. GSD's answer is a single, absolute precedence rule: **first-party always wins, and an overlay can
only add — never override.**

Here's the mechanism. The registry of first-party capabilities is frozen at build time (more on that shortly).
Installed third-party capabilities are composed *on top* of that frozen base as an overlay, and the overlay is
validated against the base before it's trusted. If an installed capability collides with a first-party one on
*anything* — the same id, the same skill or agent name, the same config key, the same command family — the
*overlay* is rejected, never the first-party capability. The reserved name prefixes (`gsd-`, `anthropic-`) are
refused outright, so nobody can publish `gsd-security` and borrow implicit trust. A single malformed or
colliding overlay is skipped with a recorded warning; it never crashes the load.

And there's a beautiful asymmetry in *how* it fails. Skipping a broken capability's **steps and contributions**
fails *open* — the loop just proceeds without an optional addition, which is safe. But skipping a broken
capability's **gate** would fail *dangerously*: a dropped gate behaves exactly as if the check had *passed.* So
GSD does the opposite — when it can't evaluate a capability that declares a gate, it **injects a synthetic
blocking gate** at that point and halts, naming the capability whose gate couldn't run. The governing instinct,
everywhere in this system: when in doubt, drop the *addition* but never weaken a *control.*

### Trust: parity without a sandbox

Third-party capabilities raise an obvious question: if an installed capability can ship its own hooks, its own
MCP servers, its own command code, isn't that dangerous? GSD's answer is unusually honest, and worth
understanding because it's a real security-design position, not a hand-wave.

The framework makes a deliberate split: it grants third parties **full artifact parity** — your capability may
ship exactly the same executable surfaces GSD Core ships — but it does **not** grant them symmetric **trust.**
And crucially, it admits there is **no sandbox.** Meaningful Node-level sandboxing is fundamentally in tension
with full parity, and the maintainers chose parity. So once you consent and install, a third-party capability
runs with the same permissions as GSD itself — exactly like any npm package you install. The protection is not
a technical cage; it's three properties:

1. **Consent** — you explicitly approved the executable surfaces *before* they ran. Installation is copy-only:
   it stages files, validates, and writes a ledger, but it *runs no code* — there's no `postinstall` equivalent,
   so merely downloading a capability can't trigger anything. Because hooks fire on the *next* tool call with no
   first-use prompt, consent is bound at install time, and the disclosure names every executable surface,
   including the exact environment and working directory any MCP server would launch with.
2. **Integrity** — the bundle you consented to is the bundle that runs. GSD computes a hash over the *entire*
   bundle — every file, deterministically — and the loader matches against it on every load. Tamper with any
   file, including a single hook script, and the capability goes inert until you re-consent. Auto-update is
   *off* by default (your last explicit trust was for version N, not N+1), and even when enabled, an update
   that changes the *set* of executable surfaces re-prompts rather than silently gaining a hook.
3. **Reversibility** — `remove` undoes exactly what was done. The ledger records every file the capability owns
   and every fragment it spliced into a shared config file, so removal deletes precisely those and strips
   precisely those fragments, leaving no orphaned state.

There's an administrator's lever over *where* capabilities may come from — a registry policy that ranges from
permissive-with-consent (the default), through a host-based allowlist, to full local-only lockdown — and,
tellingly, a *malformed* policy value fails closed. The consent record itself lives *outside* the repository,
keyed to the real project path, so a checked-out repo that ships a capability bundle can't self-activate on
someone else's machine. The whole posture is: *you know what you installed, you got what you were shown, and you
can completely undo it* — which is not the same as "it's guaranteed safe," and GSD says so out loud.

### The frozen registry

One recurring phrase above was "frozen at build time." Here's what that means. Every `capability.json` — all
thirty-three of them — is compiled, at release, into a single generated registry module that the loop reads
instead of re-scanning folders on every invocation. The generator validates each manifest, enforces the
cross-capability rules (unique names, acyclic dependencies, no config-key collisions), inlines each prompt
fragment, pre-sorts the hooks at each loop point, and emits one deterministic module partitioned every way the
loop needs it: by role, by owned skill/agent/config-key, and — the index the resolver actually reads — **by
loop point.** A committed drift guard (`--check`) rebuilds the registry in memory and byte-compares it against
the committed file; if a manifest changed and nobody regenerated, CI fails. It's the same "generated committed
artifact" pattern you'll see again in Part III-C — a single source of truth, compiled, with a guard that makes
staleness impossible to merge.

---

## 2 · Runtime capabilities — one codebase, sixteen tools

Now the same word, the other meaning. A **runtime capability** is a `capability.json` with `role: "runtime"`,
and instead of extending the loop it describes *a host GSD can install into* — Claude Code, Cursor, Codex,
Copilot, and a dozen more. The genius of the arrangement is stated in one line: **hosts are data, not code.**
Adding support for a new tool means writing a descriptor, not forking the codebase. GSD is authored once, in
Claude Code's native dialect, and four stacked layers carry it to everywhere else.

The problem is real: sixteen tools with different command syntaxes, different tool names, different agent
formats, different config locations, and — the deep one — wildly different *orchestration powers.* Some can spawn
nested sub-agents five levels deep; some can't spawn any. GSD handles all of it without a per-host fork.

### Layer 1 — the descriptor

Each host is a manifest declaring itself over a fixed vocabulary: where it keeps config, how its commands are
invoked (`/gsd-plan-phase` for most, but `$gsd-plan-phase` for Codex), where its hooks live and in what dialect,
and — most importantly — a set of *capability axes*: can it dispatch nested sub-agents, and how deep? can it run
work in the background? does it let GSD pick the model, or must GSD inject the model per-agent? does it have a
filesystem for state?

Compare two hosts and the abstraction snaps into focus. **Claude Code** declares everything at full strength:
nested dispatch five deep, background execution, a full sub-agent toolkit, filesystem state — and because it's
the *native* format, its local artifacts need *no* conversion at all. **Copilot** declares the opposite:
markdown config, no nested dispatch (`maxDepth: 1`), no declared hook events, and — tellingly — marks its
runtime axis `undocumented`, a sentinel we'll see bite in Layer 3. Between them sit hosts like **Cursor** (needs
three converters, dispatches only two deep) and **Codex** (powerful, but flat — `maxDepth: 1` — and driven
through shell-variables and TOML).

### Layer 2 — the converter

The artifacts themselves — Snip's commands, skills, and agents — are rewritten into each host's dialect by a
converter engine: a library of pure functions, one per host, that mechanically transform the neutral Claude
version into the target. Each does four rewrites: the **slash namespace** (`gsd:foo` → `gsd-foo`, or Codex's
`$gsd-foo`), the **tool names** (Claude's `Bash(` and `Edit(` become Cursor's `Shell(` and `StrReplace(`), the
**frontmatter** (rebuilt into each host's schema — Copilot even quotes the description so a leading `[BETA]`
can't crash its YAML parser), and an injected **adapter header** that teaches the host's model how to behave
(how it's invoked, which tools to use, how to spawn a sub-agent in *this* host's idiom).

This is precisely why the command↔skill 1:1 mirror from Part II exists. Because every one of the seventy
commands has exactly one neutral skill twin, conversion is a *pure per-artifact function* — read one neutral
skill, emit that host's version — rather than sixteen hand-maintained ports. "One codebase, sixteen dialects"
is a compile step, and the mirror is what makes it one.

### Layer 3 — degradation, and failing closed

Conversion handles *syntax.* The deeper problem is *capability* — what happens when a host simply can't do
something GSD assumes. This is where GSD is at its most careful. A single pure module reduces the host's
declared axes to a verdict — `full`, `degraded`, or `unsupported` — for each of six interface points:

| Interface point | Full | Degraded | Unsupported |
|---|---|---|---|
| **command** | native slash commands | palette / TOML forms | prose-only menu |
| **dispatch** | nested, depth ≥ 2, full toolkit | flat (`maxDepth 1`) → *waves run inline* | can't spawn → single-agent inline |
| **model** | host calls the model for GSD | GSD injects the model per-agent | — |
| **hooks** | host fires an event bus GSD subscribes to | engine-owned bus | rule-text instructions only |
| **state** | filesystem | sandboxed storage / append-log | — |
| **artifact** | slash-file | prose | skills become tool calls |

The concrete cases make it vivid. Copilot and Codex both declare flat dispatch, so the **wave model you saw in
the last chapter collapses**: an orchestrator that would fan a wave out to parallel sub-agents instead does each
wave's work itself, inline, in one context. A host that can't run background work can't have a detached
orchestrator, so GSD keeps it inline (the always-safe path). Every host here is "passive" on the model axis, so
GSD can't ask the host to call a model — it writes the resolved model into each agent's frontmatter instead.

And the rule that ties it together, the one to remember: **when a host's documentation is silent about an axis,
GSD assumes the floor, not the ceiling.** That `undocumented` sentinel Copilot declared? It validates, but it
never propagates — GSD substitutes the most-restrictive known value and emits a warning. A missing axis, an
unknown future value, an explicit `undocumented` — all three degrade *closed.* Silence is treated as the least
capability, never a hopeful guess at the most. That single instinct is why GSD can support a half-documented
tool without ever crashing on it: it just runs GSD in a diminished, safe mode and tells you so.

### Layer 4 — placement, and the universal escape hatch

Installation reads the descriptor (again, data not code), runs the named converters over all seventy artifacts,
and writes the `gsd-`-prefixed results into the host's config directory — flat for simple hosts, or tucked under
namespace-router skills for hosts where a flat listing would cost too many tokens. The embedded-host installer
and the command-line installer share the same engine, so an embedded install is byte-identical to a
first-party one.

Two shipped surfaces are worth singling out. The **MCP server** is the universal escape hatch: a small
JSON-RPC server that exposes just two of the six interface points — command dispatch and state I/O — which turns
out to be *enough for any MCP-speaking host to drive GSD with no bespoke plugin at all.* A tool GSD has never
heard of — VS Code, Gemini, whatever — still gets the core loop through it. And the **OpenCode adapter**
illustrates the thin-bridge pattern for a host that *does* have a plugin API: rather than re-implement GSD's
hooks, it translates OpenCode's events into Claude-shaped payloads, *spawns the existing Claude hook scripts as
child processes,* and translates their answers back. Zero hook logic is duplicated; the Claude-native scripts
remain the only implementation, reused across a process boundary.

The tally: fifteen in-tree host descriptors — three **tier-1** (Claude, Codex, Antigravity: fully tested,
merge-gated) and twelve **tier-2** (shipped, lighter automated coverage) — plus the open-ended MCP long tail.
That's the "sixteen and counting" the README promises, and none of them is a fork.

---

## 3 · Host hooks — the guardrails around the runtime

We've now met one meaning of "hook" already in this chapter — the *lifecycle hooks*, the twelve loop points
that feature capabilities attach to. This section is the **other** meaning, and it's genuinely unrelated. A
**host hook** is a small script that *your editor* runs when *its own* events fire — a file is about to be
written, a tool just ran, the session started, a compaction is imminent. Where lifecycle hooks are declarative
attachments inside GSD's workflow, dispatched on an internal bus, host hooks are imperative scripts the host
runs from outside to police the runtime. Same word; different layer, different owner, different trigger. Keep
them apart and the hook system is simple.

There are about eighteen of these scripts, each reading a JSON event on standard input and answering with either
advisory text (injected into the agent's context) or, rarely, a hard *block* that cancels the pending action.
They bind to the host's lifecycle: session start runs a canonical-path repair and a background update check;
writes into the planning folder trigger injection scanners; any substantive tool call, and every subagent-stop,
stop, and *pre-compaction*, runs the context monitor; a change to `config.json` triggers the hot-reload. A
universal contract runs through all of them: wrap everything in try/catch and **exit silently on any error** — a
broken guard must never wedge the agent — and never block except in the one or two guards deliberately designed
to.

### The guards worth knowing

- **The context monitor** solves the problem Chapter 1 hinted at: fresh sub-agents stay clean, but *the
  orchestrating session itself fills up,* and if it hits the wall it can trigger an automatic compaction that
  silently discards the very planning state it was relying on. So this hook watches the orchestrator's remaining
  headroom (warning at 35%, critical at 25%) and — firing *before* a compaction, among other moments — advises
  the agent to wrap up or checkpoint. It's advisory by design; it warns, it never seizes control.

- **The injection bookends.** Two guards defend against prompt injection from opposite directions. The
  **read-injection scanner** fires on the *output* of any read, fetch, or search and scans that freshly-ingested
  external text for injection signatures — including a clever "summarization-survival" pattern set, because in a
  long session the compressor can't tell a real instruction from a poisoned one it read out of a web page. The
  **prompt guard** fires on *writes into the planning folder* — catching injection being *planted* into the
  artifacts that later become agent prompts. Scanner catches ingestion; guard catches planting. Both advise by
  default; the scanner only escalates to a hard block if you opt into strict injection blocking.

- **The worktree path guard** is the one hard blocker in the default set, and it exists for Snip specifically:
  when Snip's phase ran parallel executors in isolated worktrees, a model under load occasionally tried to write
  an absolute path back into the *main* repo instead of its worktree. The prose rule in the executor's prompt
  gets skipped; this guard enforces it at the tooling layer — but only after *proving* it's inside a
  GSD-managed worktree, so it can't misfire on ordinary work.

- **The quiet infrastructure.** The **config-reload** hook makes a mid-session `config.json` edit take effect
  immediately, because forcing a `/clear` would destroy the continuity the whole framework protects. The
  **canonical-path bootstrap** fixes an install where the plugin manager never ran the installer, by symlinking
  the paths GSD's file-includes expect. And a background **update check** compares your installed version — and
  each hook's own version stamp — against the latest, surfacing "update available" without blocking anything.

One guard is worth calling out precisely *because it isn't a host hook.* The **package-legitimacy gate** —
GSD's defense against "slopsquatting," where an AI hallucinates a package name and an attacker pre-registers it
— lives in the *workflow* layer, not `hooks/`. It classifies every package the plan wants to install (too new?
too few downloads? no source repo?), strips the clearly-fake ones from research before they're saved, and
inserts a `checkpoint:human-verify` task that *halts execution* before installing anything suspicious. It's the
"still stops in autonomous mode" guard you met in the last chapter, and it degrades conservatively: if its
checker is unavailable, it flags *every* package rather than waving them through. It's a good reminder that
GSD's guardrails live at several layers — some in host hooks, some woven into the loop itself.

### Wiring and the syntax gate

Because every host expresses hooks differently, GSD writes each one's native config: Claude's `settings.json`,
Cursor's `hooks.json`, Codex's TOML, Copilot's inline commands, Cline's rule-text — and many tier-2 hosts carry
no GSD hooks at all, so their security surface is necessarily thinner. And there's a small, telling piece of
build discipline: every hook script is **syntax-checked in a throwaway VM before it ships,** because a duplicate
declaration once slipped into a shipped hook and broke tool calls for every user. A syntax error now fails the
build instead of reaching anyone. (That, too, is a preview of Part III-C's theme: GSD's quality machinery is
mostly a catalog of specific past scars, each turned into an automated guard.)

The organizing principle across all eighteen: **advisory by default, blocking only when certain.** The guards
that could produce a false-positive deadlock all merely advise; the single guard that hard-blocks does so only
after positively proving the dangerous condition. Defense in depth, with the safety catch always on.

---

You've now seen how GSD grows and how it travels: features that bolt onto the loop's twelve points without
editing it, a trust model that grants parity without pretending to sandbox, a four-layer abstraction that turns
"sixteen tools" into a compile step and fails closed on anything undocumented, and a layer of guardrails that
watches the runtime from outside. The two overloaded words are, at last, fully unpacked.

What's left is the machinery that keeps all of this *honest* — how GSD verifies that what got built matches what
was asked, how it researches, how it isolates parallel work, and the remarkable quality apparatus that guards
the framework's own code. That's Part III-C, the last descent.
