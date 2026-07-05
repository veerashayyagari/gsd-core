# Understanding GSD Core

### A field manual you read like a book — surface to depths and back up again

---

GSD Core — *Git. Ship. Done.* — is a framework that drives AI coding agents (Claude Code, Codex,
Cursor, Copilot, OpenCode, and a dozen more) through a disciplined loop: **Discuss → Plan → Execute →
Verify → Ship**, one phase at a time. It exists to solve a single, stubborn problem — that an AI's work
gets *worse* the longer it works, as its context window fills with its own accumulated output — and it
solves that problem with an idea you will meet on nearly every page of this book: **do the heavy thinking
in fresh, disposable sub-agents, and keep the durable truth in files on disk.**

That is the whole framework in two sentences. The rest is consequences.

This book is about those consequences — all of them, in order, at a readable pace.

---

## Who this is for

You already sense that GSD Core is large. It is: seventy commands, a hundred-odd orchestrator workflows,
thirty-four specialist agents, thirty-three capabilities, a hundred-and-forty-odd source modules, fifty-six
architecture decision records. Reading it cold is not a good time. This book is the guided path through it.

It's written for someone who wants to **understand the framework well enough to improve it** — to look at a
subsystem and see not just what it does but where it strains, what's half-built, what could be better. To get
there you first have to genuinely understand the thing, and that is the job of these pages.

## The promise

By the last page you should be able to comfortably answer any question about three things:

1. **Structure** — how the whole machine is put together, and why it has the shape it does.
2. **Experience** — what it actually *feels like* to use, step by step, in every mode.
3. **Internals** — how it works underneath: the loop, the state, the agents, the capabilities, the ports.

And, having understood all three, you should be able to **see the gaps** — the seams where the design is
under tension. We don't hand you a list of them. We walk you through the system until you can see them
yourself, which is the only kind of seeing that's any use when you go to fix them. The final part of the
book is where that sight arrives.

## How this book is organized

It has a shape: you start at the surface, you descend, and you come back up changed.

- **Part I — The Experience.** What GSD is for, and a narrated day in its company. You *use* the system.
- **Part II — The Structure.** We climb to altitude and look down: the layers, the vocabulary, the map.
- **Part III — The Deep Dive.** We open every box the map showed you. Three chapters, descending:
  the **core** (the loop, its modes, its memory), how it **extends and ports** (capabilities, runtimes,
  hooks), and its **quality and platform** machinery (verification, the build, the tests).
- **Part IV — Coming Back Up.** We reassemble the whole system in your now-expert head, walk its **seams**,
  and end with a short self-test so you can confirm you came out the other side fluent.

Then a short appendix — a glossary, a one-line digest of all fifty-six ADRs, and a "where does X live" file
map — for when you want a fact fast.

| | | |
|---|---|---|
| **Part I** | [`01-the-experience.md`](01-the-experience.md) | why it exists · a day with GSD · the front doors |
| **Part II** | [`02-the-structure.md`](02-the-structure.md) | the six layers · two words · two control surfaces |
| **Part III-A** | [`03-deep-dive-core.md`](03-deep-dive-core.md) | the loop · modes · state · scenario catalog · agents · config |
| **Part III-B** | [`04-deep-dive-extend.md`](04-deep-dive-extend.md) | capabilities · one codebase, many runtimes · hooks |
| **Part III-C** | [`05-deep-dive-quality.md`](05-deep-dive-quality.md) | verification · research · parallel work · the build machine |
| **Part IV** | [`06-coming-back-up.md`](06-coming-back-up.md) | the whole system · the seams · can you answer these? |
| **Appendix** | [`07-appendices.md`](07-appendices.md) | glossary · ADR digest · file map |

## How to read it

**Front to back, the first time.** Each part is written to be read in one sitting and stands on its own —
you only change files when you finish a part. The chapters build: Part II assumes you felt Part I, Part III
assumes you saw the map. If you already know GSD, the table above is also a jump table.

**You never need to leave the page.** This is a rule the book holds itself to. When we discuss a workflow, an
agent, a capability, or a module, we tell you *what happens inside it* right here, in the prose. You will see
file paths — always as an optional *"if you want to read the code"* aside, never as homework. You can
understand this entire book without opening a single source file. (You are, of course, warmly encouraged to
go read the code once you know what you're looking at.)

**We follow one example the whole way.** To keep the machinery concrete, the book builds a single small
project with GSD, start to finish, and revisits that same build at every depth:

> **Snip** — a tiny URL shortener. You give it a long link, it hands back a short code; visit the code and it
> redirects you. Small enough to hold in your head, real enough to have a data model, an API, a splash of UI,
> and more than one phase of work. In Part I you'll *watch* Snip get built. In Part II you'll see the *map* of
> what happened. In Part III you'll open every box Snip passed through. In Part IV you'll understand Snip's
> journey well enough to critique the machine that made it.

One last thing before we start. Two ordinary words — **capability** and **hook** — each mean two different
things in GSD, and almost every confusion about the framework traces back to that. We'll clear it up
properly in Part II. For now, just know the ambiguity is deliberate, and that once you can tell the two
senses apart, a surprising amount of the system clicks into place.

Turn the page.
