# Part II — The Structure

*In which we climb to altitude and look down at the machine.*

---

In Part I you built Snip. You typed a handful of commands, answered some questions, watched files appear in a
`.planning/` folder, and ended with a pull request. It felt — deliberately — like a conversation with a
competent colleague who happened to be very good at paperwork.

This part is about what was actually happening underneath that conversation. We're going to climb to altitude
and look down at the whole machine at once. Not to open any boxes yet — that's Part III — but to see how many
boxes there are, how they're stacked, and what the wires between them carry. By the end you'll have a map, a
vocabulary, and two mental hooks that make the rest of the framework click into place.

Keep Snip in mind as we go. Everything below is the machine that built it.

## The six layers

Every time you press Enter on a `/gsd:` command, a request falls through six layers, top to bottom, and the
results climb back up. Here is the whole stack on one page:

```
   ┌─────────────────────────────────────────────────────────────┐
   │  YOU                                                         │
   │  /gsd:execute-phase                                          │
   └───────────────────────────┬─────────────────────────────────┘
                               │
   ┌───────────────────────────▼─────────────────────────────────┐
   │  1 · COMMANDS          commands/gsd/*.md                     │
   │  Thin front doors. A command knows almost nothing; it points │
   │  at a workflow and says "run that."                          │
   └───────────────────────────┬─────────────────────────────────┘
                               │
   ┌───────────────────────────▼─────────────────────────────────┐
   │  2 · WORKFLOWS         gsd-core/workflows/*.md               │
   │  The real orchestration. Long Markdown "prompt-programs"     │
   │  that read references, mind the state, and spawn agents.     │
   └──────┬──────────────────┬──────────────────┬────────────────┘
          │                  │                  │
   ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
   │ 3 · AGENT   │    │ 3 · AGENT   │    │ 3 · AGENT   │
   │ fresh       │    │ fresh       │    │ fresh       │   agents/gsd-*.md
   │ context     │    │ context     │    │ context     │
   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
          │                  │                  │
   ┌──────▼──────────────────▼──────────────────▼────────────────┐
   │  4 · CLI TOOLS         gsd-tools.cjs  +  src/*.cts modules   │
   │  Deterministic, tested code. Does the exact, boring, correct │
   │  things: parse, compute, read-modify-write state.            │
   └───────────────────────────┬─────────────────────────────────┘
                               │
   ┌───────────────────────────▼─────────────────────────────────┐
   │  5 · FILE STATE        .planning/                            │
   │  PROJECT · REQUIREMENTS · ROADMAP · STATE.md · config.json · │
   │  phases/… — the durable truth, in plain text on disk.        │
   └─────────────────────────────────────────────────────────────┘
```

Let's walk it, top to bottom, with Snip.

**Layer 1 — Commands.** When you typed `/gsd:execute-phase`, you touched a file called
`commands/gsd/execute-phase.md`. It is astonishingly thin — a page of frontmatter and a few sentences. It
declares which tools it's allowed to use, and then it essentially says: *"load the execute-phase workflow and
do what it says."* Commands are the labelled buttons on the front of the machine. There are seventy of them,
and almost none of them contain any real logic. This thinness is on purpose; you'll see why in a moment.

**Layer 2 — Workflows.** The button is wired to `gsd-core/workflows/execute-phase.md`, and *this* is where the
work lives. GSD's workflows are not short. `execute-phase.md` alone is roughly eleven thousand words of
carefully sequenced instructions — a program written in English (with fenced shell commands sprinkled through
it) that an AI executes step by step. It's the orchestrator: it figures out what plans exist for Snip's
current phase, groups them by dependency, decides what can run in parallel, spawns workers, collects their
results, and updates the record. There are about a hundred of these workflow files, and they are the beating
heart of the framework. When someone says "how does GSD *do* X," the honest answer is almost always "read the
X workflow."

**Layer 3 — Agents.** The workflow doesn't do the coding itself. It *delegates* — spawning specialist
sub-agents, each defined by a prompt in `agents/gsd-*.md`. For Snip's redirect-endpoint phase, the
execute-phase workflow spawned a `gsd-executor` agent for each independent chunk of work, and each executor
started with a **completely clean context window** — it knew about its one job and the artifacts it was
handed, and nothing else. There are thirty-four specialists in the roster: planners, researchers, a code
reviewer, a security auditor, a debugger, verifiers, and more. You never invoke them; the workflows do. Their
freshness is the entire point, and it's the first of our two big ideas — hold that thought.

**Layer 4 — CLI Tools.** Agents and workflows both need to do things that must be *exactly right* every time —
compute the next phase number, safely update a status field, parse a roadmap. Guessing is not acceptable here,
and language models guess. So this work is handed down to real, deterministic, unit-tested code: a command-line
program called `gsd-tools.cjs` backed by around a hundred and forty source modules. When a workflow needs to
mark Snip's phase complete, it doesn't *describe* the update in prose and hope — it runs a tool.

**Layer 5 — File State.** And everything the machine knows about Snip lives, in the end, as plain text files in
a `.planning/` folder in Snip's repo: what the project is, what it requires, the roadmap of phases, the current
state, the configuration, and a subfolder per phase holding that phase's discussion, plan, and results. No
database. No server. Just human-readable files you could open, edit, or commit to git. This is the second big
idea, and we'll come to it.

That's the stack. Requests fall down it; results climb back up; the truth settles at the bottom as files.

## The one insight that explains the whole shape: Markdown orchestrates, code obeys

Look again at layers 2 and 4 and you'll notice something strange for a piece of software: **the orchestration
logic is written in English, and the deterministic logic is written in code — and the English is in charge.**

This is the single most important thing to understand about GSD's construction, so let's say it plainly. The
phase loop is *not* a `while` loop somewhere in a program. There is no function called `runPhaseLoop()`. The
loop is a **specification** — a set of Markdown workflow files that an AI reads and enacts — supported by a
**state machine and a CLI** that the AI calls to keep its bookkeeping honest. The workflows are the brain and
the intent; the code is the hands and the ledger.

Why build it this way? Because the "worker" is a language model, and the most natural way to instruct a
language model precisely, while still letting it be flexible, is a well-written document — not an API. GSD
leans all the way into that: its workflows are prompts elevated to programs. But prompts drift and hallucinate,
so anything that must be *correct* — numbering, state transitions, file placement, parsing — is pulled out of
the prose and into tested code that the prose merely *calls*. The result is a system that is simultaneously
soft where it wants to be adaptable and hard where it must be exact.

Once you internalize this — *Markdown decides, code computes* — a dozen otherwise-puzzling design choices turn
obvious. It's why the commands are thin (they're just labels for documents). It's why the workflows are so
long (they're the actual program). It's why there's a whole layer of deterministic modules underneath (to stop
the AI from freelancing on the parts that can't be freelanced). Keep this lens on for the rest of the book.

## Two words that unlock everything

Almost every real confusion about GSD comes from two ordinary words that each carry two distinct meanings. Once
you can hear which meaning is intended, the framework stops fighting you. Here they are.

### "Capability" — a runtime, or a feature

A **capability** is a self-contained folder under `capabilities/` with a `capability.json` manifest. But there
are two completely different kinds of thing wearing that name, told apart by a single `role` field:

- **Runtime capabilities** (`role: "runtime"`) describe *a host GSD can install into.* There's one for Claude
  Code, one for Cursor, one for Codex, one for Copilot, and so on — fifteen in all. A runtime capability is a
  spec sheet: where this host keeps its config, how to translate GSD's files into this host's dialect, what
  this host can and can't do. It contains no features. It's an **adapter**.

- **Feature capabilities** (`role: "feature"`) are *optional pieces of behavior you can switch on in the loop.*
  Test-driven development, deep research, security auditing, a memory palace, code review — eighteen of them.
  A feature capability plugs into the phase loop and adds or changes what happens. It's a **plugin**.

Same word, same folder, same file format — but an adapter for a host and a plugin for the loop are not remotely
the same animal, and reading `capabilities/` as if they were is the fastest way to get lost. When this book
says "capability," it will always tell you which kind it means.

### "Hook" — a host event, or a loop point

A **hook** is a place where extra behavior gets attached. Again, two unrelated systems share the word:

- **Host hooks** live in `hooks/` as little executable scripts, and they fire on *your editor's* events — when
  a session starts, before a file is written, after a tool runs. They're guardrails around the runtime: one
  watches context usage, one scans for injected instructions in fetched text, one keeps you from writing
  outside the worktree. They are imperative code reacting to the host.

- **Lifecycle hooks** are something else entirely: named *points in the phase loop* where feature capabilities
  attach. There are exactly twelve of them — `discuss:pre` and `discuss:post`, `plan:pre` and `plan:post`,
  `execute:pre`, `execute:wave:pre` and `execute:wave:post`, `execute:post`, `verify:pre` and `verify:post`,
  `ship:pre` and `ship:post`. A feature capability says "run my step at `plan:pre`" or "inject my check at
  `execute:wave:post`," and the loop obliges. These are declarative attachment points, not scripts.

Host hooks guard the *runtime*; lifecycle hooks extend the *loop*. Keep them apart and the hook system is
simple; blur them and it's baffling.

### Two smaller structural facts, while we're here

Two more facts about how the pieces are laid out, both of which will matter later:

- **Every command has a twin skill.** Alongside the seventy files in `commands/gsd/`, there are seventy nearly
  identical files in `skills/` — the same content in a slightly different wrapper. Commands are Claude Code's
  native `/slash` format; skills are a portable format that GSD can mechanically translate into whatever any
  other host expects. The skill is the neutral master copy; the command is one rendering of it. (This is how
  one codebase serves sixteen different tools — the machinery is Part III's problem, but the 1:1 mirror is a
  structural fact worth filing away now.)

- **The code you run was compiled from code you can't see in git.** The deterministic modules live in `src/` as
  TypeScript (`.cts` files). They're compiled — at publish time — into the `.cjs` files that actually execute,
  and those compiled files are deliberately *not* checked into git. So if you go looking for `state.cjs` in the
  repo you won't find it; you'll find `state.cts`, its source. This "build-at-publish" arrangement trips up
  everyone once. Now it won't trip up you.

## Two control surfaces

The last thing on the map isn't a layer — it's the two dials that change how the whole machine behaves. You
met both, lightly, while building Snip. Here they are as structure; Part III takes them apart.

**`config.json` — the dials.** Sitting in Snip's `.planning/` folder is a small JSON file that governs how the
loop runs: which optional features are on, which models each kind of agent uses, how git branches are named,
whether a second AI reviews the code, and much more. It follows a rule worth memorizing now because it's
everywhere in GSD: **absent means enabled.** If a feature flag isn't in the file, it defaults to *on*. You
don't switch defaults on; you switch them *off*. The config file is small precisely because most of what it
could say is already true by omission. In Part III we'll go option by option through what each dial does to the
flow.

**Execution mode — the tempo.** The same workflow can run two ways. In **interactive** mode — the way you built
Snip — the machine stops to ask you things and waits at the gates: it interviews you during Discuss, pauses for
your sign-off before shipping. In **autonomous** mode, it doesn't stop: it makes reasoned assumptions instead
of asking, chains one phase into the next, and only halts when it genuinely needs a human or hits the end.
There are lighter tempos too — quick one-offs, fast fixes — but interactive-versus-autonomous is the big fork,
and the *same* workflow file drives both. How it bends from one to the other is one of Part III's headline
acts.

---

You now have the map: six layers, one governing insight (*Markdown orchestrates, code obeys*), two double-edged
words (*capability*, *hook*), and two control surfaces (*config*, *mode*). That's enough altitude.

Part III is the descent. We're going to open every box on this map — starting with the one in the middle, the
loop itself, and how it bends between its modes — and we're not coming back up until we've seen the bottom of
each one.
