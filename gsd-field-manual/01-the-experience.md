# Part I — The Experience

*In which we meet the problem, and then spend a day solving it.*

---

## 1 · Why this exists

Every AI coding session starts clean. The model reads your request, thinks, and answers. If that were the
whole story, no one would need a framework — you'd just ask, and receive.

But a session is almost never one exchange. You ask a follow-up. You paste an error. You say "no, not like
that." You course-correct when the model drifts. Each turn adds text to the **context window** — the finite
buffer of tokens the model can attend to at once — and that is where the trouble starts.

### The quiet failure

As the window fills, something subtle happens, and the reason it's dangerous is that it happens *quietly.* The
model doesn't crash. It doesn't warn you. It keeps answering in the same confident tone. But the answers get
worse. The constraint you stated in the first message is now buried under twenty exchanges of accumulated
detail, competing for the model's attention against everything that came after it. Researchers have a name for
this: **context rot.**

You have almost certainly seen its symptoms without having a word for them:

- The model contradicts a decision it explicitly agreed to ten minutes ago.
- The code style drifts away from the conventions it followed perfectly at the start.
- A new plan quietly ignores a requirement you stated clearly — but stated a long time ago.
- It hallucinates a function signature it had exactly right earlier in the same conversation.

None of this is a bug, and that's the important part. It is a fundamental property of how transformer attention
works over long sequences. The model was never "remembering" in the human sense; it was weighting relevance
across a fixed window. As that window fills with the accumulated noise of a long session, the signal-to-noise
ratio falls, and the quality of everything falls with it. By the time an agent is writing the fifth file of a
big task, it may already have lost the constraint it was given in the first message.

### The trap

The obvious escape is to start over — hit `/clear` and open a clean window. And that does cure the rot. But it
trades rot for amnesia: now you have to re-explain the project, re-paste the relevant files, re-state the
constraints, re-establish everything you'd built up. You are stuck between two bad options. Keep going, and
quality decays. Start fresh, and continuity dies.

The entire design of GSD Core is an escape from that trap. Hold onto this framing, because everything else in
this book is a consequence of it.

### The bet

Here is the move. GSD's central insight is that **most of the work in a coding session doesn't need to happen
in your main context at all.** Research, planning, writing code, checking it — each is a discrete, bounded job.
Each can be handed to a *separate* sub-agent that starts with a clean, carefully scoped window, does its one
task at full strength, and then disappears.

This is not a trick to delay context rot. It is a structural way to avoid it. An executor writing the tenth
file of a four-hundred-line plan doesn't degrade — because it isn't carrying the other nine files' worth of
conversation. It started fresh, read only what its task required, and it operates at full capacity precisely
*because* it never accumulated the cruft. The rot never gets a chance to form.

But fresh context on its own would be useless — a brilliant worker with amnesia. Three more ideas make it
work, and they fit together deliberately:

- **Thin orchestrators.** Your main session — the one that coordinates everything — must itself never do heavy
  lifting, or it would rot like any other long session. So it doesn't touch source files. It spawns agents,
  collects what they produce, updates a shared record, and moves on. Because it does so little, its own context
  grows slowly. (Even this has limits, and GSD watches its own headroom — a theme we'll return to.)

- **File-based state.** When a sub-agent finishes, its work can't just live in a conversation that's about to
  be thrown away. So every step writes its output to a **file** — plain Markdown and JSON on disk. The rule is
  blunt and load-bearing: *agents do not rely on memory; they rely on the file.* This is what makes fresh
  context survivable. It's also what lets you `/clear`, close your laptop, come back in three days, and have
  the next agent pick up from durable artifacts instead of a reconstructed memory.

- **Spec-driven work.** A fresh agent given vague instructions produces vague output — fresh context isn't the
  same as *good* context. So before any code gets written, the framework produces structured specifications:
  what was decided, what the research found, what the plan is. By the time an executor touches a file, it works
  from a precise brief, not a re-interpretation of a long chat.

Put together: fresh context makes each agent think clearly; specs make it think about the *right* thing; and
carefully engineered agent prompts — GSD calls this meta-prompting — make it think about that thing *well,*
without you having to re-teach it every time.

### The loop, as a set of guards

All of this is organized into a repeating five-step loop — **Discuss → Plan → Execute → Verify → Ship** — and
you'll live through it in the next chapter. The thing to understand now is *why* it has steps at all. The loop
is not ceremony. Each step exists to guard against a specific way things go wrong that no earlier step could
catch:

- **Discuss** guards against planning on wrong assumptions. Decide *how* to build the thing, not just *what,*
  before a planner starts guessing — because it will guess plausibly and wrongly, and you'll find out only
  after the work is done.
- **Plan** guards against executing a broken design. This is the moment ambiguity is most expensive: several
  parallel executors, each quietly making different assumptions about the same thing, produce conflicts. A plan
  gets checked *before* execution, not after.
- **Execute** is where context rot is actually defeated — each executor runs fresh.
- **Verify** guards against shipping work that missed the brief. A phase isn't done because execution finished
  without errors. It's done because what was built is what was planned, and what was planned is what was
  decided.
- **Ship** closes it: the pull request, the archived artifacts, the record updated. Then the loop turns again.

### The honest cost

It would be dishonest to sell this as free, and GSD's own documentation is candid about the price, which is
worth repeating because it tells you when *not* to reach for the loop:

- **Overhead.** Running discuss, plan, and execute as separate steps takes more elapsed time than typing "write
  this feature" into a plain chat.
- **Latency.** Spawning fresh sub-agents is slower than a single in-context edit, and while one runs, its work
  is invisible until it returns.
- **Ceremony.** For a typo, a renamed variable, a missing import, the full loop is overkill.

The rule of thumb falls right out of that: **if a task could be fully specified in one short prompt and finished
in one agent turn, skip the loop.** If it needs research, touches files you haven't read recently, or depends
on decisions that aren't settled yet, the loop is protecting you. (GSD even ships lighter side-doors for the
small stuff — you'll meet them by the end of the next chapter.)

Two ideas run underneath all of it, and if you remember nothing else from this chapter, remember these:
*front-load cheap effort — a conversation, a spec, a plan check — to avoid expensive rework later;* and *never
let signal drown in accumulated noise.* Every design choice in the rest of this book is one of those two ideas,
wearing a different hat.

Enough theory. Let's watch it work.

---

## 2 · A day with GSD

We're going to build something small and real: **Snip,** a URL shortener. You hand it a long link; it gives
back a short code; visit the code and it redirects you. It's tiny enough to hold in your head and real enough
to have a data model, an API, and more than one phase of work. We'll take Snip from an empty folder to a merged
pull request, and I'll show you exactly what you type, what happens, and what you see.

*(A note on spelling: GSD's commands have two working forms — `/gsd:new-project` and `/gsd-new-project` are the
same command; the colon is canonical, the hyphen is an alias the tutorials use. I'll use the colon form.)*

### Install

In Snip's empty directory:

```
npx @opengsd/gsd-core@latest
```

The installer asks you two things. First, **which runtime** — which AI coding tool you're driving. Claude Code
is the headline choice, but Codex, Cursor, Copilot, Windsurf, OpenCode and a dozen others are all supported; we
pick Claude Code. Second, **global or local** — install into your home directory for all projects, or into just
this one. We pick local. A moment later:

```
✓ Installed 86 skills to .claude/commands/
✓ Installed agents to .claude/agents/
✓ GSD Core ready — run /gsd-new-project to start
```

A `.claude/` folder now exists, holding the commands you'll type, the sub-agents they'll spawn, and the GSD
runtime itself. One practical note the tutorial insists on: relaunch your session with permissions relaxed
(`claude --dangerously-skip-permissions`) so the sub-agents don't stop to ask you before every file write.
Otherwise the fresh-context machinery spends its life waiting on you.

### Starting the project

```
/gsd:new-project
```

This is a real conversation, and it has a shape. First a banner tells you where you are —

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► QUESTIONING
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

— and then it asks, plainly, in an open text box rather than a menu: **"What do you want to build?"**

This is the one input that sets everything in motion, so if you want to follow along and watch the flow with
your own eyes, this is the moment to paste in a real description. There's no required format — a paragraph of
plain intent is exactly right. For Snip, you might paste:

> *Snip — a URL shortener. A small HTTP service in Node.js: I `POST` a long URL and get back a short code;
> visiting `/{code}` redirects to the original link. Mappings must persist across restarts; codes are generated
> automatically and are short and unique; later on I want to count how many times each link is visited. No user
> accounts for now — keep it minimal.*

That's all it needs to begin. And now GSD does the thing that makes the difference: it *interrogates* you,
gently. It challenges vague words, makes the abstract concrete, surfaces assumptions you didn't know you were
making, hunts for the edges. Do short codes expire? Can a user pick a custom code? What happens when a code doesn't exist — a 404, or
a redirect home? You're not filling in a form; you're being helped to think. When it believes it understands,
it stops and asks permission to write things down:

```
Ready?  →  [ Create PROJECT.md ]   [ Keep exploring ]
```

Choose *Create PROJECT.md* and it writes your project description to disk, with your requirements sorted into
what's validated, what's still a hypothesis, and what's explicitly out of scope.

Next it asks how you like to work, and turns your answers into a configuration file. The choices each come with
a recommended default: run in **YOLO** mode (auto-approve and go) or **Interactive** (confirm at each step); how
coarse or fine to slice the work; whether to run phases in **parallel**; whether to track everything in git;
which **models** to use; and whether to switch on the optional guards — pre-execution research, a plan checker,
a post-execution verifier, a drift guard. Say yes to sensible defaults and move on; we'll return to every one of
these dials in Part III, because they quietly reshape everything that follows.

Then a fork worth noticing: **"Research the domain ecosystem before defining requirements?"** Say yes and GSD
spawns four researchers *in parallel* — one each for the stack, the features, the architecture, and the common
pitfalls — and a minute later you have five research notes on disk. For something as well-worn as a URL
shortener you might skip it; for unfamiliar territory it's a cheap way to not start naive. Either way, you then
turn your requirements into a numbered list — every version-one capability gets a requirement ID, which will
matter later when the verifier checks that each one was actually built.

Finally, the roadmap. GSD spawns a specialist — the roadmapper — which proposes how to slice Snip into phases,
and shows you the result right in the conversation:

```
## Proposed Roadmap
3 phases | 7 requirements mapped | All v1 requirements covered ✓

| # | Phase                    | Goal                               | Requirements |
| 1 | Link store & create API  | Persist links, mint short codes    | REQ-1, REQ-2 |
| 2 | Redirect endpoint        | Resolve a code to its target (302) | REQ-3, REQ-4 |
| 3 | Click analytics          | Count visits per code              | REQ-5, REQ-6 |
```

And it stops at a gate: **Approve / Adjust phases / Review full file.** *Adjust* sends your notes back to the
roadmapper and it tries again; *Approve* commits the roadmap, writes a project instruction file so your AI tool
knows the house rules, and you're initialized:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PROJECT INITIALIZED ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## ▶ Next Up — [SNIP] URL shortener
Phase 1: Link store & create API — persist links, mint short codes
/clear then:
  /gsd:discuss-phase 1   — gather context and clarify approach
  /gsd:plan-phase 1      — skip discussion, plan directly
```

Notice that last panel. GSD almost always ends by telling you exactly what to run next. You are rarely left
wondering.

### The folder that appears

Before we take a phase around the loop, look at what `/gsd:new-project` left behind, because this folder *is*
the project's memory:

```
.planning/
  PROJECT.md        what Snip is; requirements split into validated / active / out-of-scope
  REQUIREMENTS.md   a requirement ID for every v1 capability
  ROADMAP.md        the phases, their goals, their requirement mappings; phase 1 marked "pending"
  STATE.md          where we are right now — the session's memory across resets
  config.json       the working preferences you just chose
  research/         the five research notes (only if you said yes to research)
```

Every one of those files is plain text you can read, edit, or commit. And every step of `new-project` committed
its own artifact as it went — so if your session had died halfway through, nothing would have been lost. That's
file-based state from Chapter 1, made concrete. The conversation is disposable; the folder is the truth.

### One turn of the loop

Now the heart of it. We'll take Phase 1 — Snip's create-a-link API — all the way around. The ritual is to
`/clear` before each phase (fresh context, remember) and let the files carry continuity. Everything a phase
produces lands in its own folder, `.planning/phases/01-link-store/`.

**Discuss** — `/gsd:discuss-phase 1`. The framework loads what it already knows, quietly scouts your codebase,
and works out the *phase-specific* open questions — not generic "think about UX" prompts, but the real gray
areas for this phase, skipping anything already decided earlier. It lets you pick which ones are worth its time,
then goes a few questions deep on each. For Snip's create API it might ask: how is a short code generated —
random, or a hash? How long? Where do the mappings live — a file, SQLite, Postgres? What's the response when
someone posts a URL that's already been shortened? Scope creep ("ooh, and analytics!") gets caught and parked in
a deferred list rather than derailing the phase. The output is a single file, `CONTEXT.md`, whose *Implementation
Decisions* section is precisely what the planner will read next.

> There's a second way to run this step, and it's worth knowing now because we'll dissect it in Part III. Instead
> of interviewing you (the default), Discuss can run in **assumptions mode**: a sub-agent reads a handful of your
> actual files, forms a set of assumptions — each tagged *confident,* *likely,* or *unclear,* each with the file
> that justifies it and a note on what breaks if it's wrong — and asks you only to confirm or correct. Same
> `CONTEXT.md` out the other end; a quarter of the questions. Codebase-first instead of interview-first.

**Plan** — `/gsd:plan-phase 1`. Four researchers again fan out in parallel and leave a `RESEARCH.md`. Then the
planner reads your decisions and the research and writes the work as **atomic task plans** — small, independent
units, each in its own numbered file (`01-01-PLAN.md`, `01-02-PLAN.md`, …). Each plan names the exact files it
will touch, the steps to take, and — crucially — a **verify command** the executor can run to prove the task
worked. Before any of it is saved, a *plan checker* reads each plan and asks a hard question: will this actually
achieve the phase's goal? If not, it sends it back. Nothing broken gets to execution if the checker can help it.

**Execute** — `/gsd:execute-phase 1`. Now the parallelism pays off. GSD groups the independent plans into
**waves** and, for each plan in a wave, spawns a fresh executor with its own clean 200k-token window. Each does
its one task, runs its verify command, and commits atomically. What you see is calm:

```
Wave 1 (parallel):
  [Executor A] → 01-01-PLAN.md  link store (read/write mappings)   ✓ committed
  [Executor B] → 01-02-PLAN.md  POST /links endpoint               ✓ committed
[Verifier] checking codebase against phase goals…
  REQ-1 persist a link ✓   REQ-2 mint a unique code ✓   Status: PASS
```

Each executor leaves a `SUMMARY.md`; the run leaves a `VERIFICATION.md` recording which requirements are
covered. (One guard never sleeps, even here: if a plan would pull in a package that looks suspicious, execution
*stops* and asks a human — GSD will not silently install a flagged dependency. More on that safety gate in Part
III.)

**Verify** — `/gsd:verify-work 1`. This is a conversation, not an interrogation: one check at a time, plain
answers, and anything a passing automated test already proves is quietly skipped so you're only asked about what
genuinely needs a human eye.

```
[1/2] POST a long URL to /links — do you get back a short code?  > yes
[2/2] GET that code — does it 302 to the original URL?           > yes
Both checks passed. Phase 1 verified.
```

If something fails, GSD doesn't shrug — it diagnoses the root cause and writes *fix plans,* which you execute in
a targeted "gaps only" pass and then re-verify. The record of every check and its outcome lands in `01-UAT.md`,
which survives a `/clear` and feeds any gaps back into planning.

**Ship** — `/gsd:ship 1`. A preflight, then a push, then GSD writes the pull-request body *from the planning
artifacts you've been accumulating* — a summary, the changes, the requirements addressed, how it was verified,
the key decisions — and opens the PR:

```
Pull request created: https://github.com/you/snip/pull/1
Title: feat(phase-1): link store & create API
```

That's the whole arc — empty folder to merged code — for one phase. Snip's phases 2 and 3 are the same ritual
again. The loop is a rhythm: each step is easy because the one before it did its job.

### "What now?" — the router

You don't have to remember any of this sequencing. At any point you can type `/gsd:next` (or its zero-friction
cousin `/gsd:progress --next`) and GSD will look at the state on disk, figure out where you are, and tell you —
then just *do* — the single right next thing:

```
## GSD Next
Current: Phase 1 — Link store & create API | 100%
▶ Next step:  /gsd:discuss-phase 2
   Phase 1 is shipped; Phase 2 (Redirect endpoint) is up.
```

It also refuses to march you off a cliff: if there's an unresolved checkpoint, an error state, or a failed
verification you haven't dealt with, it hard-stops and says so rather than blithely advancing. The router is the
closest thing GSD has to a "just keep going" button.

### The other tempo — autonomous

Everything above was **interactive** — GSD stopped and asked, and waited for you. But the *same workflows* can
run **autonomously.** Type `/gsd:autonomous` (or add `--auto` to the router) and GSD stops waiting: it walks
every remaining phase in order, running discuss → plan → execute on each, re-reading the roadmap between phases
in case new work got inserted, and finishing the milestone on its own.

```
 GSD ► AUTONOMOUS ▸ Phase 2/3: Redirect endpoint  [████░░░░] 45%
```

What changes is mostly *the asking.* Where interactive Discuss interviews you, autonomous mode runs a **smart
discuss** that proposes answers to each gray area — with rationale, in a batch — and, left fully to itself,
accepts its own well-reasoned recommendations. The roadmap-approval gate, the "ready?" confirmations, the
per-check verification prompts: all skipped when the machine is confident.

But — and this is the reassuring part — the things that *protect* you do not turn off. The plan checker still
runs. The package-legitimacy gate still stops for a suspicious dependency. And when GSD hits something it
genuinely can't decide, or finds real gaps, it *pauses and asks* rather than guessing:

```
Phase 2 (Redirect endpoint) needs a decision:
  Should unknown codes 404, or redirect to the home page?
  [ Answer ]   [ Skip this phase ]   [ Stop autonomous mode ]
```

When it pauses like that in an unattended run, it can drop a small `WAITING.json` marker on disk — a
machine-readable "I need a human, here's the question" — that an external watcher or orchestrator can notice,
answer, and clear, so the run continues. That little file is how GSD hands off between "running by itself" and
"needs you" without losing its place. Autonomous mode is the right tool when the phases are well-understood; when
design decisions are still unsettled or the work is genuinely novel, you're better off discussing first.

### The side doors

Not everything is a phase, and GSD knows it. Around the main loop sits a family of front doors for other
situations — you'll meet their internals in Part III, but here's when you'd reach for each:

- **`/gsd:new-milestone`** — the sequel to `new-project`. When Snip's v1 is shipped and you're starting v1.1, this
  loads the history, gathers what's next, and rolls a fresh set of phases — *continuing* the numbering.
- **`/gsd:quick`** — real GSD guarantees (a planner, an executor, fresh contexts, an atomic commit) but no phase
  overhead, for a fix or small feature that doesn't belong on the roadmap. When you're unsure whether something
  is trivial, this is the safe default.
- **`/gsd:fast`** — genuinely trivial edits, done inline with no sub-agents at all: a typo, a config value, a
  `.gitignore` line. It self-guards — if the job turns out to be bigger than it looks, it stops and sends you to
  `quick`.
- **`/gsd:spike`** — throwaway experiments to *learn* something before you commit to a plan: does this library even
  work the way I think? It produces verified knowledge, not shippable code.
- **`/gsd:sketch`** — throwaway *design* exploration: two or three quick HTML mock-ups of a direction, to compare
  before you build the real UI.
- **`/gsd:map-codebase`** — the front door for an *existing* project. It sends four mappers across your repo and
  writes seven notes about your stack, architecture, conventions, testing, and trouble spots — so that when you
  then run `new-project`, GSD asks about what you're *adding* and already knows what you have.

### You've used it

That's a day with GSD. You installed it, described a project, watched it interrogate you into clarity, take a
phase from decisions to a plan to parallel execution to verification to a pull request, and you saw it do the
same thing on its own when you let it. You met the folder that remembers everything and the router that always
knows what's next.

You've been the *user.* You've felt the surface. What you haven't seen is the machine under the surface — how a
thin command becomes a hundred parallel decisions, where those fresh agents actually come from, how the same
workflow file bends between interactive and autonomous, what every dial in `config.json` really does.

That's the descent. Turn to Part II, and we'll climb up to see the whole map before we start opening boxes.
