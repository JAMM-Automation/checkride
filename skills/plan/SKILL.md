---
name: plan
description: One entry point for any task bigger than a single edit, from reconnaissance to close. Five steps in order: recon with one round of questions, a plan in a single task file, the user's approval, staged execution with a check after every stage, and a close where every done-criterion is proven by a command. Depth scales with size — medium tasks get a self-written plan and a self-run skeptic pass; large ones (production, money, data, publishing, migrations, multi-session work) also get the skeptic agent, the architect agent when there is a real design fork, and the validator agent before close. Use for new subsystems, migrations, irreversible or cross-component changes. Not for routine runs or content made by an existing process.
argument-hint: [short task description]
---

# /checkride:plan — a task from recon to close

One entry point, five steps, one task file. The steps are the same for every task; only the depth changes with size. The user steps in at two or three points: answers to blocking questions, approval of the plan, and a "yes" to close a large task.

> **The skill is the rails; the task file is the state.** Instructions fade after a few hundred tool calls; the task file is re-read before every stage. Rely on the file, not on your memory of this text.

**Project profile.** First step: read `.claude/checkride.md` at the repo root (`git rev-parse --show-toplevel`). No profile → work with the defaults in `${CLAUDE_PLUGIN_ROOT}/templates/checkride.md` and say so in the first line of your reply ("no project profile — using defaults; copy the template to `.claude/checkride.md` to customise"). Paths below in `‹angle brackets›` come from the profile or those defaults.

## Size — decide first

| | Small | Medium | Large |
|---|---|---|---|
| Sign | one edit or action from a known pattern, undo with git | several files or one component, reversible, fits one session | any of: production, money, database, publishing, new subsystem, migration, several sessions, shared config every session loads (`CLAUDE.md`, hooks) |
| Task file | none: three lines in chat — what I do · how I check · the proof | one, ≈30 lines | one, ≈60 lines; bulky material goes into attachments |
| Plan written by | — | me | me; `checkride:architect` if there is a real design fork or the session runs on a weaker model than the architect's |
| Skeptic | my own three lenses | my own three lenses | `checkride:skeptic`, once |
| Approval | no | yes | yes |
| Verification | a command with output | criteria with proof | the same + `checkride:validator` if production changed or something irreversible happened; otherwise I run the "what could be missed" pass myself |

Small → just do it without this skill. Unsure between two sizes → take the bigger one. Size is the first line of the task file; the user can change it at approval.

## 1. Recon

**Guards — silently, before the first question:**
- A routine run or content from an existing process → this skill and the validator do not run; use the existing skill directly.
- Out of scope for this repo (`## Guards` in the profile) → one redirect sentence from the profile and **stop**.
- A plan on this topic already exists (`ls ‹plans› | grep -i <topic>` + grep ‹trackers›) → **update it, do not create a second one**.

**I read the facts myself:** the relevant source of truth, data schema, existing processes, current texts and settings. Asking the user for anything Read/Grep/Bash can find is not allowed.

**One round of questions** — only those whose preconditions are already settled; a question that depends on another open one goes to the next round:

```
❓ **Q1** — **<title>**: <question; where useful, 2–3 options with consequences>
➡️ <recommended answer + why>
```

- **Every question carries a recommendation** — the user should be able to answer "1 — yes, 2 — your option, 3 — no".
- **An option in a question is a claim about the current setup:** before proposing a tool (a role, a service, a script), check the registries named in the profile.
- **Questions go in the body of the reply as text**, not in a form.
- **No questions that block the plan → skip the round** and bring non-blocking ones to approval together with the plan.
- A third round means the task is probably two tasks — say so. More branches than one round can hold, or the user asks to dig in first → offer `/checkride:grill` in one line and run it after a "yes", then step 2.

## 2. Plan

The task file `‹plans›/<slug>.md`, from `references/task-template.md`: goal and requirements, the user's decisions, stages with checks, done-criteria, rollback. No separate log — status and proof live in the same tables.

- **Criteria and stage checks** follow `references/checks.md`: a fact a command can refute, with the expected output written before the run.
- **Executors** are a column in the stages table, per `references/executors.md`: small work inline; an agent only where a cheaper model does the same job or it isolates heavy reading.
- **Alternatives** only when there is a real fork: the chosen one goes into the stages, the rejected one with its reason into "Changes and cuts". No fork → write nothing.
- **Architect** (large, under the condition in the size table): run `checkride:architect` with the task description, the user's answers, the recon facts and the profile path. It is read-only and returns the task-file text; I write it verbatim.
- **Skeptic** (large): run `checkride:skeptic` with the path to the task file and **without my chain of reasoning**. CONCERNS → apply the fixes. STOP → correct or justify; at most one more round. Medium: I run the three lenses myself — simpler? fragile assumptions? reuse of what exists?

## 3. Approval 🔴 point of no return

Show: "before → after" in one block, the stages, the done-criteria, the skeptic's verdict (if any) in one or two lines, open questions with recommendations. **Up to this moment only the task file has changed.** Wait for an explicit "go"; the user's edits go into the file **before** starting. Large: after "go" copy `‹plans›/<slug>.md` to `‹plans›/<slug>/approved.md` — a frozen reference for the final check.

## 4. Execution

For every stage:

1. **Re-read the task file.**
2. Run the stage with the executor from the table. Before starting an agent or a separate session, read `references/executors.md` §3 (brief) and §7 (launch and acceptance).
3. **Run the stage check** — myself, on the artifact: an agent's report is not proof. FAIL → find the root cause, confirm it with a command before fixing, one line in "Changes and cuts". Second FAIL in a row on the same stage → before the third attempt, a fresh subagent with no session context and a mandate to REFUTE my hypothesis. Three attempts at most, then back to the user.
4. **Update the task file:** only the status and proof in the stage row — the row's content is not rewritten after approval; a change goes into "Changes and cuts". "Handed over to someone else / belongs to the next stage" in an agent report → a line in "Open", closed only after checking what the receiver actually produced.
5. **A one-line checkpoint to the user, facts only:** `Stage 2/5 — database. PASS: 1 row. Next: the /invite command.` For agent stages also tokens: actual vs forecast.

**A separate "yes" every time — plan approval does not replace it:**
- an irreversible action inside a stage: production, database, deploy, money, publishing;
- merging a worktree branch into the main branch — check the stage on the branch first;
- a token-heavy run: 10+ parallel subagents or other large fan-out, even if it is in the approved stages. A stricter rule in the repo's `CLAUDE.md` or profile wins.

**The plan is alive — edits go into the file immediately:**
- The user changed the requirements → stages and criteria in **the same session** + a line in "Changes and cuts" (a removed criterion too, otherwise the validator flags it as a lost requirement). Agents already running → `references/executors.md` §7.
- A rejected option → "Changes and cuts". Anything temporary (a file, a disabled guard, a test record) → "Open" right away, cleaned up before close.
- **Do not silently extend a list the user approved** — an extra field is a question to the user.
- Run the skeptic again only if a load-bearing requirement changed.
- **Writing a checker or an assertion → first read `references/checks.md` § "Traps in grep and done-checkers".**

## 5. Verify and close

1. **Check the criteria.** Never state "done" — give proof for each criterion: command output, not a retelling.
   ```
   Planned N criteria, closed M:
   ✅ D1 <criterion> — <command> → <output>
   ❌ D3 <criterion> — not closed because <...>
   ```
   Anything open → back to step 4.
2. **Validator** (large, production changed or something irreversible): run `checkride:validator` with the criteria, the task-file path and the profile — **without my execution report**. FAIL → step 4 and validate again. NOT-VERIFIABLE → to the user for a manual check. Every "what could be missed" finding is either worked through or dismissed with a line in the task file. Large without production → no validator, but I go through its four "what could be missed" classes myself (lost requirements, consequences of additions, references to cancelled things, forgotten traces), one command each, result as a line in the task file.
3. **No monitoring for days.** The deadline is the first real run of what changed; the check happens once. Whatever cannot be seen before close → one line in ‹backlog› with a trigger, and the task closes.
4. **Close.** "Open" is empty or moved to the backlog → commit your files per `## Saving` in the profile. Large: "N of N criteria confirmed. Close?" and wait for "yes". Then `/checkride:save` — after the "yes" for large tasks, or at the end of the session for medium ones; it collects the lessons; it is not run while a FAIL is open. The task file stays in ‹plans› as the source of truth; to resume, read the file and continue from the first row that is not ✅.

## Anti-patterns

- ❌ Running it for routine runs or content from an existing process
- ❌ Asking the user what you can read yourself · a question without a recommendation · executing before approval
- ❌ "Done" without proof per criterion · giving your own report to the validator · saving while a FAIL is open
- ❌ More process than the task's risk pays for: retelling context in the task file, criteria written twice, a checker for a text edit, an agent where inline is enough, "just in case" monitoring stages
- ❌ Hard-coded repo paths in this file — only from the profile. Keep it under ~250 lines; details live in `references/`

## References (read when needed)

- `references/task-template.md` — the task file template (plan and state together)
- `references/checks.md` — how to write criteria and checks, what counts as proof, how much checking is enough
- `references/executors.md` — choosing an executor and model, briefing an agent, launch and acceptance
- `${CLAUDE_PLUGIN_ROOT}/agents/{architect,skeptic,validator}.md` — the other participants of the loop
- `${CLAUDE_PLUGIN_ROOT}/templates/checkride.md` — the project profile and its defaults
