# Part IV — Coming Back Up

*In which we reassemble the whole system in one view, walk the seams where it strains, and confirm you came out
the other side fluent.*

---

You went in at the surface and you reached the bottom. You've seen the loop that isn't a loop, the chain of
files that carries the state, the fresh agents and the waves they run in, the dials that bend it all, the
plugins that extend it and the descriptors that port it, the verifier that reasons backward, and the rigor the
framework turns on its own code. Now we climb back up — and the climb is not a recap. It's the moment the pieces
lock together, because you finally have the vocabulary to see them cooperate. Then we'll do the thing this whole
book was building toward: walk the seams, the places the design is under tension, so you can see for yourself
where GSD could be better.

---

## 1 · The system, whole

Let's build Snip one more time — the same journey from Part I, but now every box is open, and we'll name what's
actually happening at each turn.

You type `/gsd:new-project` and paste a paragraph about a URL shortener. What you're really doing is entering the
**command layer** — a thin file that does nothing but hand off to a **workflow**, a long prose program the model
now interprets. That workflow interrogates you not to fill a form but to *widen the spec* before any assertion
has to be checked, and it writes what it learns to `PROJECT.md`, `REQUIREMENTS.md`, a roadmap, a `STATE.md`, and
a `config.json` — the `.planning/` folder that is the project's durable memory. Every one of those writes is a
deterministic tool call, not the model free-handing text, because anything that must be *correct* is pushed down
to code while the *judgment* stays in prose. That single split — **Markdown orchestrates, code computes** —
is the shape of the whole machine.

Now a phase. You `/clear` — deliberately, because continuity doesn't live in the conversation, it lives in the
files. Discuss reads the folder back, surfaces the phase's real gray areas (or, in assumptions mode, reads your
code and proposes answers), and writes `CONTEXT.md`. Plan reads *that* — never the conversation — spins up
researchers whose findings are cached by content hash and tiered by confidence, then a planner that writes work
as atomic plans, then a checker that refuses to pass a plan that won't hit the goal. Each plan carries its
**must-haves**: the truths, artifacts, and wirings the verifier will later demand, seeded from the roadmap's
success criteria and allowed only to *grow.* This is the artifact chain — **CONTEXT → PLAN → SUMMARY → UAT** —
and it is the real control flow: each step begins in a clean context and reconstructs everything it needs by
reading the previous step's file.

Execute is where the fresh-context bet cashes out. The orchestrator groups independent plans into **waves** and
spawns an executor for each — handing it *paths, not contents,* so the executor spends its own clean window
reading exactly what it needs while the orchestrator stays lean. On Claude those executors run in isolated git
**worktrees**, and — the invariant that makes parallelism safe — they may write their own summaries but never the
shared `STATE.md` or `ROADMAP.md`; when the wave ends the orchestrator becomes the single writer, merges through a
careful gauntlet, runs the tests, and only *then* records progress. A plan is never marked done while the
integration is red.

Then the verifier, reasoning backward from the goal under an adversarial stance, treating the executor's own
summary as a *claim* rather than evidence, checking that Snip's redirect doesn't merely contain the string "302"
but is actually wired to issue one — and abstaining honestly on the behavior it can't prove from static text.
Its verdict routes the loop: gaps send you to a targeted replan; a clean pass with nothing left for a human
sends you to a coverage-aware UAT that only asks what the tests couldn't settle; and ship assembles the pull
request *from the artifacts you've been accumulating all along.* The PR isn't summarized from memory. It's
composed from the chain.

And underneath all of it, the parts you learned to see in Part III were cooperating the whole time. `config.json`
was bending the flow at every step — which agents spawned, which model each used, which gates stopped you.
Optional **capabilities** were attaching at the loop's twelve points (TDD tagging tasks, the memory palace
filing decisions), extending the loop without editing it. The **runtime abstraction** was standing ready to run
this same journey on Cursor or Codex, converting the artifacts and degrading gracefully where a host can't
nest. And a layer of **guards** — the worktree path guard keeping executors in their lanes, the injection
scanners watching what got read and written, the package-legitimacy gate refusing to install a hallucinated
dependency — was policing the runtime from outside, advisory by default, blocking only when certain.

Stand back and the load-bearing ideas are all one idea, restated: **context rot is the enemy, and files are the
cure.** Fresh context keeps each worker sharp; the artifact chain lets fresh workers hand off without memory;
file-based state survives the `/clear` that makes fresh context possible. Around that spine, a consistent set of
instincts: *front-load cheap effort to avoid expensive rework* (discuss, plan-check, probes); *absent means
enabled,* so the careful path is the default path; *first-party always wins,* so extensions can only add;
*verifier reach equals spec reach,* so you widen the question rather than sharpen the judge; and *fail closed
everywhere* — an undocumented runtime axis, an unevaluable gate, a malformed config, an ambiguous lock — resolves
to the safe answer, not the hopeful one. That last instinct, more than any feature, is the character of the
system. GSD would rather stop and tell you than guess and be confidently wrong.

That's GSD, whole. Now, where does it strain?

---

## 2 · The seams

No system this ambitious is without tension, and the honest way to end a deep read is to name where the design
is load-bearing against its own limits. None of what follows is a takedown — each seam has a real mitigation, and
several are *documented by the framework itself.* But each is also a place where the residue is real, and
therefore a place where improvement work would matter. This is the gap-sight the whole book was for; treat it as
a map of where to look, not a verdict.

**The two words are a permanent tax.** "Capability" means an adapter *or* a plugin; "hook" means a host script
*or* a loop point. The framework works hard to keep them straight — a discriminator field, dedicated
disambiguation in the docs, even a lint rule against drift — but the overloading is a genuine cognitive cost that
every new contributor and every reader pays, and that no amount of documentation fully removes. It's the clearest
candidate for a naming refactor, and also the one most expensive to change now that both meanings are woven
through the code, the manifests, and the ADRs.

**The crown jewels have the thinnest adversarial net.** GSD's most load-bearing code is its state machine, its
phase logic, and its verifier — and by the framework's own admission, roughly *half* of the core library's
lines, including parts of exactly those modules, are currently excluded from mutation testing. The unit tests
exist; it's the *antagonistic* tier — the one that proves the tests would actually catch a broken line — that
hasn't reached them yet. There's a stated plan to bring one module into scope per release, which is the right
posture, but the fact remains: the code you'd least want to be wrong is, today, the code with the least proof
that its tests would notice.

**The orchestrator still rots.** Fresh sub-agents solve context rot for the *workers,* but the orchestrating
session itself accumulates and can, at the limit, trigger an automatic compaction that discards the very planning
state it was steering by. The context monitor watches for this and warns — but it is, in the framework's own
words, a *signal, not a guarantee.* The core thesis has a residual hole at its own top layer: the coordinator is
mitigated against rot, not immune to it. A reader looking for the deepest architectural gap should look here.

**"Runs on sixteen tools" hides a steep gradient.** The cross-runtime abstraction is genuinely elegant, but the
experience it delivers is uneven and mostly *invisible* to the user. On a tier-2 host with flat dispatch, the
parallel wave model silently collapses to inline execution; some hosts carry no security hooks at all, so the
guardrail layer is simply thinner; and the fail-closed handling of an undocumented axis, while safe, means a
half-documented tool quietly runs GSD in a diminished mode the user may never realize they're in. The promise is
kept in the letter — it *runs* — but "runs" spans a wide range, and the framework surfaces that range only
faintly.

**The most important logic is the least checkable.** The workflows are the engine, and they are enormous prose —
eleven thousand words in the case of execute — interpreted by a model. Byte budgets protect the model's
attention, and everything that can be made deterministic has been pushed into tested code. But the orchestration
*judgment* that remains in prose cannot be unit-tested the way a function can; it can only be validated by
running it and watching. That's an inherent tension of the meta-prompting approach, not a bug — but it means the
correctness of the single most central layer rests on prose plus a model's reading of it, which is the hardest
thing in the system to pin down.

**Verifier-reach relocates the problem rather than closing it.** "Widen the spec, not the verifier" is a strong
principle, and the calibration data behind it is convincing. But it moves the reliability question one step
upstream: now correctness depends on the *probes'* recall — a missed probe category is still a missed edge — and
on humans, because the genuinely irreducible cases are handled by *abstain-and-flag,* which is honest but pushes
the hardest correctness calls onto a person. The framework has made the residue smaller and more visible, which
is real progress; it has not made it vanish, and it doesn't claim to.

**Consent is not a sandbox.** The third-party trust model is admirably honest — consent, integrity,
reversibility — but the framework says plainly that there *is* no sandbox: a capability you consent to runs with
full permissions, exactly like any npm package. The protections guarantee that you knew what you installed, that
it wasn't swapped, and that you can fully remove it — provenance and undo, not runtime safety. As the ecosystem
of third-party capabilities grows, that procedural-not-technical barrier is a seam worth watching.

**The generated surface is power with a maintenance bill.** The "generate it, commit it, guard it against drift"
pattern is one of GSD's best ideas — a single source of truth, mechanically enforced. But it also means a large
committed surface of generated files that must all stay in sync, and a contributor who edits a source without
regenerating hits a wall of failing checks. It's a deliberate, well-defended tax; it is still a tax, and it
grows with the surface.

Read those eight together and a theme emerges that is itself worth stating: **GSD's gaps are mostly at the edges
of its own best ideas.** Fresh context is brilliant — except the orchestrator. File-based memory is the cure —
except it's an enormous prose surface to maintain. Verifier-reach is a genuine advance — except it leans on probe
recall and human judgment. Port-by-descriptor is elegant — except the experience is uneven. This is what a mature
system's seams look like: not crude failures, but the honest residue of ambitious choices. That residue is your
starting map.

---

## 3 · Can you answer these?

Here is the test the book set itself. If you can answer these from what you've read — no source files, no docs —
you came out the other side fluent across the three competencies. Each carries a pointer to where it was covered,
so this doubles as a review index.

**On structure — how it's put together:**

1. What are the six layers a `/gsd:` command falls through, top to bottom? *(Part II)*
2. Why is there no function called `runPhaseLoop()` — where does the loop actually live? *(Parts II, III-A)*
3. "Capability" and "hook" each mean two different things. What are the four meanings? *(Parts II, III-B)*
4. What is the artifact chain, and why is it — not any program — the loop's real control flow? *(Part III-A)*
5. Why is the compiled code not in git, and what is "build-at-publish"? *(Parts II, III-C)*

**On the experience — how it feels to use:**

6. What does `/gsd:new-project` actually produce, and what do you type to start it? *(Part I)*
7. What changes when the *same* workflow runs autonomously instead of interactively — and what pointedly does
   *not* change? *(Parts I, III-A)*
8. Which front door do you use for a one-line typo, for a small fix that shouldn't touch the roadmap, and for the
   next version of a shipped project? *(Parts I, III-A)*
9. What does the `/gsd:next` router do that saves you from memorizing any of the seventy commands? *(Parts I, III-A)*

**On the internals — how it works underneath:**

10. How does a wave run several executors in parallel without them corrupting each other or the shared state?
    *(Part III-A)*
11. Explain "verifier reach = spec reach," and why sharpening the verifier doesn't help. *(Part III-C)*
12. How does one Claude-native codebase run on a tool it was never written for — and what happens when that tool
    can't do something GSD assumes? *(Part III-B)*
13. What does "absent = enabled" mean, and why did the framework choose it? *(Parts II, III-A)*
14. Where does GSD admit its own design is weakest, and why is that admission a feature? *(Parts III-C, IV)*

If those came easily, you have what this book promised: you can speak to how GSD is *structured,* how it's
*experienced,* and how it *works* — and, having seen the seams, where it could be *better.* That last one is the
real graduation. Understanding a system well enough to use it is one thing. Understanding it well enough to
improve it is another, and it's the one you came here for.

The appendix that follows is for reference — a glossary, a one-line digest of every architecture decision on
record, and a map of where each part of the framework lives — for the day you go from reading about GSD to
changing it.
