# Part III-C — Quality and Platform

*In which we see how GSD proves that what got built is what was asked, how it gathers and remembers what it
knows, how it isolates concurrent work — and, finally, how the framework holds its own code to a standard.*

---

The core turns; it extends and ports. The last thing to understand about GSD is how it stays *honest* — and
honesty here has four faces. There's the honesty of **verification** (did we build what we said?), the honesty
of **knowledge** (how does the machine know things, and how sure is it?), the discipline of **isolation** (how
does concurrent work not corrupt itself?), and — the one that will surprise you — the extraordinary rigor GSD
turns on **its own source code.** We'll end this chapter, and the whole descent, by looking at the machine that
builds the machine, which is exactly where Part IV's "seams" begin to show.

---

## 1 · Verification — did we build what we asked?

Start with the failure this whole subsystem exists to prevent, because it's subtler than "the tests fail." When
Snip's redirect phase finished, an executor could have reported success with a handler that returns a hard-coded
`302 /` for every code — task complete, goal missed. GSD's founding distinction is exactly this: **task
completion is not goal achievement.** A file can exist, compile, and be "done" while the thing you actually
wanted isn't there. Catching that gap is what verification is for.

### The one idea: verifier reach = spec reach

Here is the principle the entire subsystem is organized around, and it's genuinely worth internalizing because
it's counterintuitive. A verifier works *backward* from the goal, checking assertions. **But it can only check
an assertion that exists — and an assertion only exists for something that was written down.** So the
verifier's reach is bounded by the *spec's* reach. The way to make verification more reliable is therefore not
to make the verifier smarter; it's to **widen the spec** so there are more assertions to check.

GSD's designers didn't assume this; they measured it. Asked to catch defects that can't be *deduced* from the
requirement text alone — the genuinely non-inferable cases — the verifier caught **zero of twelve**, and, worse,
was *confidently* wrong while missing them (it sat around 93% confidence on answers that were incorrect). The
sharp consequence: because it's high-confidence *and* wrong on exactly the cases that matter, **no confidence
threshold can separate its hits from its misses.** You can't tune your way out. And a verifier asked to grade
its own work against a *restated* goal simply rationalizes a pass. The conclusion is structural: don't sharpen
the judge, and never let it grade its own narrative — grade against *explicit predicates the spec supplies,* and
on the truly irreducible cases, have it **abstain and flag** rather than guess. That single change — from the
model's own confidence to an external predicate — dropped the false-pass rate on the hardest cases from 100% to
17%. Once an omitted edge *is* written into the spec, by the way, the verifier then catches it 94–100% of the
time. The verifier was never the problem. The missing predicate was.

Everything else in this section is that idea wearing different hats.

### The engine: goal-backward, and adversarial

The verifier reasons backward through three questions — *what must be TRUE for the goal to hold? what must
EXIST for those truths? what must be WIRED for those artifacts to function?* — and checks each against the
actual code, under a deliberately adversarial stance: **assume the goal was not achieved until the code proves
otherwise,** and treat the executor's own `SUMMARY.md` as a record of what it *claimed,* never as evidence.

The assertions it checks — the **must-haves** — come from two places: the roadmap phase's success criteria (the
non-negotiable contract) and the plan's own declared must-haves (truths, the artifacts that back them, and the
key links that wire them together). And there's a rule that is the spec-reach principle enforced in data:
**must-haves may ADD but never SUBTRACT.** If the roadmap named five success criteria and the plan lists three,
all five are still verified. The plan can't quietly shrink the goal.

The checks go deeper than "does the file exist." An artifact is examined at four levels — it *exists,* it's
*substantive* (not a stub), it's *wired* (imported **and** used), and its data actually *flows* (an artifact can
pass the first three and still be **hollow** — wired up but fed by a hard-coded empty value). The verifier's own
rule of thumb is striking: **80% of stubs hide at the wiring level** — the pieces are all present, they're just
not connected. And for truths that depend on *runtime behavior* grep can't see — Snip's redirect must actually
issue a 302, not just contain the string "302" — presence is marked *necessary but not sufficient*
(`PRESENT_BEHAVIOR_UNVERIFIED`) until a single named behavioral test exercises it and passes. One test, run
precisely; never the whole suite, never a started server.

### The verdict, and how it gates

A phase's verification resolves to one of three words — **`passed`**, **`gaps_found`**, or **`human_needed`** —
by the most-restrictive rule first: any failed truth or missing/stub artifact or broken link means
`gaps_found`; otherwise any item needing a human eye (including every behavior-unverified truth) means
`human_needed`; only if *everything* is verified and nothing needs a human does it reach `passed`. That last
gate is enforced, not incidental: a clean headline score can never be claimed on symbol-presence alone.
`gaps_found` routes straight back to a **gaps-only** replanning pass — the next verify run checks the failed
items fully and gives the already-passed ones a quick regression glance, rather than re-verifying everything.

One asymmetry is worth pausing on because it recurs across GSD's design. The *decision-coverage* check — did
every decision from the discussion actually make it into a plan? — is a **hard block on the plan side** (fix it
now, where fixing is cheap) but merely **advisory on the verify side** (a warning that can't wedge an autonomous
run). Same check, opposite blocking semantics, chosen deliberately: block where it's cheap to comply, advise
where blocking would strand a hands-free run at 3 a.m.

### UAT that respects your time

When verification lands on `human_needed`, the leftovers become a conversational acceptance test — one question
at a time, plain answers, the transcript saved to a `UAT.md` that survives a `/clear`. The thing that keeps it
from being tedious is **coverage-aware auto-pass**: anything a passing automated test already proves is skipped,
and you're asked only about what genuinely needs judgment. The rule behind that skip is stated in the code as two
mantras — *"lenient parse, strict auto-pass"* and *"fail-safe asymmetry."* The asymmetry is the important
half: a false negative just shows you a redundant question (annoying, harmless), but a false positive would
**ship the very bug UAT exists to catch** — so auto-pass demands a non-empty, all-green, no-human-judgment
evidence list and refuses to fire on anything less. The predicate underneath is fail-closed all the way down:
absence, ambiguity, or a malformed marker all read as *not passed.* There is no vacuous green.

### The probes: widening the spec before it's checked

If reliability comes from a wider spec, *something* has to widen it — and that's the probe family, which runs at
the very front of the loop, during the spec phase, before any code exists. Two probes do the widening on two
different axes:

- **The edge probe** attacks the *shape* of a requirement. From a closed taxonomy — boundary, adjacency, empty,
  encoding, ordering, precision, idempotency, concurrency — it raises the specific edges an author tends to omit
  ("half-up or half-to-even?"; "bytes, code points, or grapheme clusters?"). Crucially, a requirement it can't
  classify is **flagged for review, never silently dropped** — the un-probeable case is exactly the blind spot
  the probe exists to surface.
- **The prohibition probe** attacks the *must-NOTs* — the things a feature could silently *become* that the
  author would never want but never thought to forbid. A shape taxonomy is useless here, so it uses adversarial
  elicitation followed by a precision pass that boils the raw list down to the two or three genuinely bespoke
  prohibitions (routine security canon — OWASP, path traversal — is deferred to the dedicated security phase, so
  the list stays sharp).

Each surfaced finding gets *resolved* into the spec — an edge becomes a pass/fail criterion (`covered`), or a
held-out test if the right behavior can't yet be stated (`backstop`), or is `dismissed` *with a mandatory
reason* (the reason is the audit trail; a wrong dismissal is precisely the silent failure being prevented). The
gate is soft — you can always write the spec anyway, with the open items stamped as explicit assumptions — but
even in fully automatic mode, **it will never auto-dismiss:** dismissing always requires a human reason. And the
payoff is the principle closing its own loop: *a resolved finding becomes a unit the goal-backward verifier
actually checks,* extending its reach to a boundary the requirement prose never stated.

### Guards at the edges

A few specialized capabilities harden the residue. **Nyquist** audits test-coverage adequacy by *generating a
real, failing behavioral test and running it* — never editing the implementation to make a test pass (an
implementation bug is escalated, not papered over). **Schema-gate** injects a mandatory schema-push into plans
that touch an ORM, defending against a nasty false green where types come from config rather than the live
database and verification passes on a lie. And **plan source-grounding** catches a planner that cites a symbol
which doesn't exist — an invented decorator, a renamed flag — by checking cited symbols against an *out-of-band
source of truth: the real code.* Its verdict is deliberately three-valued (`verified` / `missing` / `unknown`),
refusing the trap of treating "I couldn't check" as "it's fine," and it only *hard-blocks* from tools that can
actually *prove* a symbol absent, downgrading to a gentle "please confirm" for the cheap grep-level checks that
false-positive on dynamic code.

The whole subsystem reduces to one sentence: **you don't make the judge smarter; you make sure the judge was
handed every question worth asking.**

---

## 2 · Knowledge — how the machine knows things, and how sure it is

GSD's memory of the *world* (as opposed to its memory of the *project,* which is `.planning/` state) is a set of
file-backed stores, each fronted by a small deterministic seam and filled by sub-agents. One design instinct runs
through all of them: **code owns the policy and the caching; the agent owns the fetch — and the agent always
returns a file path, never raw content.** That's what keeps research from flooding the orchestrator's context.

**Research** is a provider waterfall chosen by question *kind*: documentation questions try a docs pipeline, open
web questions try a web pipeline, and a known URL goes to a scraper — with a free built-in search as the
always-available floor and one provider (Jina) as the universal last resort. Which paid providers are live is
auto-detected from *your* keys (an environment variable or a key file), so the framework quietly uses whatever
you've got. Every result is cached by a content hash, in two tiers — a cross-project tier for durable curated
docs (a library digest fetched for one project is reused in the next) and a project-local tier for web results —
with time-to-live scaled by *confidence*: a high-confidence official-docs answer is trusted for a month, a
low-confidence scrape for a day. And confidence isn't a vibe; it's a table crossing the provider's *authority*
with a *legitimacy* verdict, which is the same slopsquatting check from the last chapter feeding back in. When a
phase needs research, one researcher fans questions out and writes a single `RESEARCH.md`; when a *project* needs
it, four researchers run in parallel (stack, features, architecture, pitfalls) and a synthesizer merges them —
the same fan-out-and-merge shape as everything else in GSD.

Three more stores layer on top. **Intel** is an opt-in queryable index of your codebase — a handful of JSON
files mapping files to roles and symbols to signatures — that can *ground* planning: instead of grepping the tree
to check whether a symbol exists, the plan guard queries the pre-built index. **Graphify** is an opt-in knowledge
*graph* of the project, its edges tagged by confidence, rebuilt in the background after commits (and honest about
its own staleness — a day-old graph tells the planner to "treat relationships as approximate"). And **learnings**
climb over time: each finished phase distills its decisions, lessons, patterns, and surprises to a file; a
cross-project store dedups them by content hash so a lesson from project A can surface in project B; and a
*graduation* step promotes anything that recurs across enough phases into the project's permanent canon. On top of
all of it, the memory-palace capability adds a genuinely *temporal* graph — facts carry validity windows, and
when a later decision supersedes an earlier one, the old fact is *invalidated with an end-date, not deleted,* so
history stays queryable.

One accuracy note worth making, because the name misleads: the module called the "embedding adapter" has nothing
to do with vector embeddings — it's about *embedding the GSD engine into a host.* There is no vector search in
the core. Retrieval everywhere is deterministic — content hashes, substring queries, confidence tiers — and
anything genuinely semantic is delegated to an external tool behind a seam. The knowledge fabric is a file tree
and some JSON, kept deliberately dependency-free.

---

## 3 · Isolation — three ways to run in parallel

GSD says "parallel" about three different things, and they're easy to confuse because all three lean on git and
all three keep concurrent work from colliding. They differ in *what* they isolate, and for *how long*:

| | **Worktree (execution)** | **Workspace** | **Workstream** |
|---|---|---|---|
| Isolates | one executor's working files | a whole environment + its own `.planning/` | planning state only |
| Scope | one plan, inside one wave | a project or set of repos | one concern-area of a milestone |
| Lifetime | seconds — merged back and deleted | days/weeks — you remove it | a milestone — archived when done |
| Driven by | the harness, automatically | you, explicitly | you, explicitly |
| Extra checkout? | yes, ephemeral, per executor | yes, one per repo | **no** — same tree, different subtree |

You already met **worktrees** without naming them: they're how a wave runs several executors at once, each in its
own throwaday git worktree on a temporary branch, merged back through a careful gauntlet (right branch? no stray
deletions? summary rescued? clean tree?) and then deleted. Their one sharp edge is the **exit-42 base mismatch**:
Claude forks those worktrees from the repository's *default* branch, not your current `HEAD`, so if your branch
is ahead — an unmerged milestone branch — the plan files the executor needs simply aren't there, and a guard
halts with a loud "base mismatch" rather than doing the wrong thing. The fix is a one-line setting
(`worktree.baseRef: head`) that forks from `HEAD` instead; and even without it, GSD *auto-degrades to sequential*
and finishes the phase rather than failing. It's the soft, self-healing counterpart to the hard halt.

A **workspace** is the heavy isolation: a genuinely separate environment under your home directory — a worktree
or a full clone per repository, each on its own branch, with its own wholly independent `.planning/` — for
multi-repo efforts or risky, long-lived experiments you want quarantined from main. A **workstream** is the light
isolation: *no* extra checkout at all, just a namespaced planning subtree (`.planning/workstreams/<name>/`) so you
can plan and execute several concern areas of one repo — Snip's API and Snip's dashboard, say — concurrently
without one area's `STATE.md` overwriting another's. Workstreams even carry their own `config.json` that
deep-merges over the root (the config layering from the last chapter), and their "active" pointer is
*session-scoped,* so two terminals open on Snip can each sit in a different workstream without interfering. The
rule that keeps all three coherent is a single line buried in the worktree seam: *a local `.planning/` always
wins over worktree remapping* — which is exactly why a workspace's planning stays its own while a bare execution
worktree transparently maps back to the main tree's.

---

## 4 · The machine that builds the machine

Now turn the lens around. Everything so far has been GSD's rigor applied to *your* project. The last thing to
see is the rigor GSD applies to *itself* — because it's unusual, and because its honest gaps are where Part IV
begins.

**It compiles at publish, and generates half of itself.** Recall from Part II that the deterministic code is
written in `src/` and compiled to the `.cjs` that actually runs, which isn't checked into git. Around that sits a
striking pattern: a whole family of *generated, committed* artifacts — the capability registry, the loop-host
contract, the skills that mirror the commands, the surface inventory, the package identity — each produced from a
single source by a generator, and each guarded by a `--check` mode that rebuilds it in memory and byte-compares
against the committed copy. Change a `capability.json` and forget to regenerate, and CI fails. The framework
refuses to let its own derived files drift from their sources — the same "single source of truth, mechanically
enforced" instinct you saw in the capability registry and the runtime descriptors, applied everywhere.

**Its test discipline is a catalog of past scars.** GSD's testing standard is unusually opinionated, and every
rule traces to a specific way it once got burned. Tests must be *deterministic* — an injectable clock instead of
real timing, because a race-based lock test once flaked 40% of the time. There's an *antagonistic* tier —
property-based tests plus **mutation testing gated at 80%** on pull requests, because a mutant (a deliberately
broken line that the tests should have caught but didn't) once survived for weeks. Assertions must target
*typed surfaces* — the machine-readable JSON output and exported registries — never rendered CLI text or a grep
of the source, because asserting on source text produces tests that pass without proving anything. And bad tests
aren't quarantined; they're *deleted and replaced in the same change.* These rules are enforced by sixteen custom
lint rules that fall into two tell-tale clusters: one for test rigor (no source-grep assertions, no tautologies,
no wall-clock sleeps) and one for **Windows portability** (no fragile line-splitting, no hard-coded `/tmp`, no
bare shell-outs) — and that second cluster is a fossil record of exactly where cross-platform pain lived.

**And it is honest about its own blind spots** — which is the detail that earns trust and the one that hands
Part IV its first thread. The mutation-testing config carries a frankly-labeled *known blind spot*: nearly half
of the core library's lines — including parts of the very state, phase, and verification modules this book spent
chapters admiring — are currently *excluded* from mutation coverage, with a stated policy to bring one module
into scope per release as coverage is added. The test suite carries another: a large block of older regression
tests is grandfathered under a ratchet that can only shrink. These aren't hidden; they're written down, with a
plan. That's the posture of the whole framework turned on itself — and it's the honest place to start looking for
where the design still strains.

Fittingly, GSD keeps its own institutional memory the same way it tells you to keep yours: a single large,
machine-greppable `CONTEXT.md` at the repository root — a ledger of domain definitions, defect anti-patterns,
executor-failure classifications, and a running session log. The framework dogfoods its own core thesis. Its
memory, too, lives in a file.

---

You've now completed the descent. You've seen the loop and how it bends, its memory and its dials, how it extends
and ports, how it proves its work, how it knows things, how it isolates them, and how it guards its own code. You
went in at the surface, and you came all the way to the bottom.

Now we climb back up. In Part IV we reassemble the whole system in one view — the running example, retold end to
end now that every box is open — walk the **seams** where the design is under tension (the blind spots it admits,
and a few it doesn't), and end where a reader who came this far should end: able to answer any question about how
GSD is built, how it feels, and where it could be better. That last part is the point of the whole book.
