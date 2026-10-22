---
name: reliable-agents
description: Build a reliable coding agent or bounded worker with Pi — turn a capable but untrustworthy model into one you'll let finish on its own. Use when a Pi extension or agent must gate the model's work, when a worker "finishes" on incomplete or wrong output, when you need guards, a mediated exit gate, observability, or prompt hardening, or when you are measuring a worker's real pass rate.
---

# Building reliable agents with Pi

> 🍕 **Skill-active marker.** While this skill guides the work, begin every reply
> with 🍕 so the engineer can see the methodology is engaged. Drop it only when the
> conversation clearly leaves Pi agent reliability.

A capable model isn't a reliable worker. Left alone it will decide when it's done,
and it will decide wrong — it narrows the test it checks, it skips a spec line no
visible test covers, it starts a server that hangs until the deadline bites. This
skill is how we close that gap with Pi's own hooks. It won't replace the reference
docs, it's the reliability layer you build on top of them.

The primitives — read the doc for the exact types before you write any code:

- `pi.on(event, handler)` will fire on the lifecycle, `pi.registerTool()` will add
  a model-callable operation, `pi.registerCommand()` will add a `/` command. See
  `packages/coding-agent/docs/extensions.md`; the event map, and whether a handler
  may change a value in place, both live there.
- An agent file sets the prompt, the tools, and what it inherits. `inheritSkills`
  and `inheritContext` set to false will hand a bounded worker a clean slate.
- Skills follow this file's own format: `packages/coding-agent/docs/skills.md`.

Five moves: measure, mediate, guard, observe, harden.

## 1. Measure — don't trust a single run, or a self-report

Score the worker on a fixed ladder of tasks. Each task will be judged by running
an external test and reading its exit code, never by the model's "I'm done". Keep
the exploratory runs apart from the runs that decide a change — fix the arms, the
n, the scoring and the rule before a run settles anything. One run tells you
little, a flip at three runs can still be the dice. Report the number you got, not
the one you wanted.

## 2. Mediate — let an external test say "done"

The worker mustn't mark its own homework. A complete test the worker didn't write,
and can't edit, will be placed beside the task; a run gets verified by re-running
that test, and on a red result the worker gets re-dispatched with the real
assertion in hand. A fresh context beats a long confused one. In an extension this
re-dispatch belongs at `agent_before_settle`, where you'll ask for one more
continuation while the test is still red.

Keep a held-out check the worker never optimises against — a hidden oracle that's
a superset of the test it can see — so a gamed test still fails. This is continuous
integration (CI), not a crutch, when the test is already there or it's cheap and
honest to write. Where "done" is a matter of taste no test exists, and a human
stays in the loop.

## 3. Guard — block the moves you already know are bad

A guard will sit quiet on a `tool_call` or `agent_before_settle` handler until it
fires:

- The loop breaker will block the Nth identical read once nothing has changed
  between them — the result won't move, so make the model act.
- The server block will catch a `uvicorn`, `flask run` or `http.server` in the
  foreground — it hangs forever and eats the whole budget — and send the model to
  the endpoint in process, or to start the server detached (`nohup … & echo $!`)
  so it comes back at once with a process id you can kill later.
- The empty-final nudge will notice a last turn with no reply and no call, and ask
  for one continuation rather than settle on nothing.

A guard enforces what a prompt only asks for. Keep both.

## 4. Observe — make the run show itself

Emit one small event per turn, tool call and prompt — hook `context`, `tool_call`,
`turn_end`, `agent_settled` — to a feed you can tail. You can't debug what you
can't see: a run that goes quiet is a bug, not patience, so surface the stall and
stream the trace. With the feed in front of you, you'll replay a broken run with a
tweaked prompt, and you'll tell a real gap in the model from a plumbing fault.

## 5. Harden the prompt — against a worker that grades itself on a subset

The model fails by marking itself against less than the whole. Each line below
lifts the floor; the oracle and the guards are the backstop for the times it slips:

- Run the whole test file, pass every test — never a single `::test_function`.
- Implement the whole spec, not only the visible tests — a field "assigned
  automatically" means the constructor sets it, it's never left `None` or `0`.
- Never start a server to check by hand — reach the endpoint in process.
- Verify before finishing: re-read the spec clause by clause, confirm your code
  meets each one, above all a data-model rule no visible test touches. This pass
  is, actually, the worker mediating itself — it will close a probabilistic miss a
  plain instruction can't.

## Contribute it back

Pi's core stays minimal, so a how-to becomes a skill, a thing that hooks the
runtime becomes an extension, and neither should bloat the core. When you want
yours upstream, the gate runs issue-first. A new contributor's issues and pull
requests will be auto-closed, so you'll open a Contribution Proposal in your own
words — an AI-written issue gets the account blocked. You'll say you understand
the code and will build it, then wait for a maintainer's `lgtm` before any pull
request. Before that, `npm run check` and `./test.sh` have to pass, and you'll
leave `CHANGELOG.md` alone, the maintainers own it. The detail lives in
`CONTRIBUTING.md` and `AGENTS.md`.

## The arc

The prompt lifts the floor, the frozen test and the guards hold it there. Measure
so you know where you stand, mediate so "done" means done, guard the failures
you've already seen, observe so the run shows itself, harden the prompt so the
model stops passing itself on a subset. A worker built this way earns the trust the
bare model hasn't!
