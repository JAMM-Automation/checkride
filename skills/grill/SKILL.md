---
name: grill
description: A hard, structured interrogation of a new task before any action — a decision tree, questions in rounds, until no branch is left silently assumed. Ends with "Shared understanding + Done criteria". Run it only after the user explicitly agrees to an offer to grill the task, or on their direct request. Domain branches come from the project profile.
argument-hint: [short task description]
disallowed-tools: AskUserQuestion, Task, Agent
allowed-tools: Read, Glob, Grep
effort: high
---

# /checkride:grill — interrogation before work

Goal: pull out of the user what they **did not think** to put into the prompt. Do nothing until the picture is complete.

Adapted from the grilling approach and the `grill-me` skill by Matt Pocock ([mattpocock/skills](https://github.com/mattpocock/skills), MIT — see `THIRD-PARTY-NOTICES.md`). The mechanics are shared by every project; domain branches and guards come from the project profile (`.claude/checkride.md`, sections `## Guards` and `## Grilling`; no profile → `${CLAUDE_PLUGIN_ROOT}/templates/checkride.md`).

## Round 0 — guards (before the first question, silently)

1. **Does this belong to this repo?** Compare with `## Guards` in the profile. A match → the redirect sentence from the profile and **stop**. Do not grill a task you are not allowed to do here.
2. **Is it worth grilling at all?** One edit in a file, a known skill, a mechanical action, content within an existing format — just do it. Grilling is for: a new feature, a new pipeline or product direction, a migration, an irreversible action, a task with a fuzzy goal.
3. **What already exists?** Quickly: tracker and plan files (`## Trackers`), the relevant source of truth, `Grep` in the domain file of the area (`## Lessons`). If the task already has a plan, grill the **delta**, not from scratch.

## Mechanics — a decision tree, in rounds

A task is a **tree**: each decision branches into the decisions that depend on it.

The **frontier** is every question whose preconditions are **already settled** — the ones you can ask now without guessing answers you have not heard yet.

1. Compute the frontier.
2. Ask **the whole frontier in one round**, numbered.
3. **Wait for the answers.** Do not work.
4. The answers push the frontier outward — compute it again; next round.

A question whose answer depends on another question that is **still open** in this round belongs to the **next** round. Mixing levels of the tree is the main mistake.

Usually 2–4 rounds. A fifth round means the task most likely splits into two — say so.

## Question format

```
❓ **Q1** — **<short title>**: <question; where useful — 2–3 options with consequences>

➡️ <your recommended answer + why exactly this one>
```

A recommendation is **mandatory for every question**. The user should be able to answer "1-yes, 2-your option, 3-no" instead of writing an essay. A question without a recommendation pushes your work onto them.

Questions go **as text in the chat**. `AskUserQuestion` is disabled here (see `disallowed-tools`).

## You find the facts yourself

**Finding facts is your job, never theirs.** If a question lacks a fact from the environment (what is in the database, what the config says now, whether this workflow exists, what the source of truth says) — go and read it with Read/Glob/Grep. Ask only for **decisions**, never for data you can get.

Subagent fan-out is disabled here (`disallowed-tools: Task, Agent` — the subagent tool has had both names) to protect the token budget. If a fact really needs heavy research: ask the rest of the frontier now, and ask permission for a research agent for that one branch separately.

## What must be covered

Do not let go until each item is either clarified or deliberately dropped. Shared branches are below; domain branches (platform, accounts, compliance, funnel…) come from `## Grilling` in the profile — cover them too:

- **End state** — what does the world look like when the task is done? What will the user be able to do that they could not before?
- **Why now** — what triggered the task. Often the real problem is not the one named.
- **Boundaries** — what is **not** included. The cheapest way to avoid extra work.
- **Reuse** — what already exists that covers this. If a workflow, skill, agent or script does 80%, that changes the task.
- **Irreversibility** — what cannot be rolled back here. Production, money, public posts, databases, external communication.
- **Money / legal** — if the task touches pricing, payouts or public content → is the repo's review step needed.
- **One-off or permanent** — a single action, or does it become a system (then it needs a source of truth, a schedule, monitoring).

## Finale — and only after confirmation

When the frontier is empty, output **one block**:

```
## Shared understanding
<3–6 lines: what we do, why, in what scope>

## Out of scope
<what we deliberately do not do>

## Done criteria
- [ ] <a check with pass/fail — a command, a query, a concrete observable state>
- [ ] ...

## Risks / irreversible
<what can go wrong and where the point of no return is>
```

**Done criteria are machine-checkable where possible** (a command, an SQL query, a URL returns 200), not "works well". They are the input to the "Verify" step.

Then — **stop and wait for explicit confirmation**. Do not start work under any circumstances.

## After confirmation

- **Large task** → `/checkride:plan`: it decides the depth (skeptic, architect, validator) by size. The chain: grill → plan → approval.
- **Repo-specific work** → where to hand it over is in `## Grilling → After the interrogation` of the profile.
- **Medium** → a task file `<plans folder>/<slug>.md` per `/checkride:plan` + a line in the tracker, then execute.
- **Small** → just execute, keeping the Done criteria as a checklist.

## What not to do

- Do not ask what you can read yourself.
- Do not ask without a recommendation.
- Do not ask "tell me more about the task" — that is not a question, it is surrender. A question is always concrete, with options.
- Do not ask next-level questions together with the current level.
- Do not start working "while they are answering".
- Do not grill forever: empty frontier → finale.
- Do not publish or mutate anything during grilling. At all.
