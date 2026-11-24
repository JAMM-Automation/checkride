# Executors — rules for the "Plan" and "Execution" steps of `/checkride:plan`

> The goal is speed and token economy during execution. Every stage gets an executor, a model, an effort level and a mode **before** the skeptic and approval, so the unnecessary parts can be cut.

## 1. Inventory (one command, first lines of descriptions only)

```bash
python3 - "$(git rev-parse --show-toplevel)" "${CLAUDE_PLUGIN_ROOT}" <<'EOF'
import sys, glob, re
root, plug = sys.argv[1], sys.argv[2]
for base, pat in [(root, ".claude/skills/*/SKILL.md"), (root, ".claude/agents/*.md"), (plug, "skills/*/SKILL.md"), (plug, "agents/*.md")]:
    for f in sorted(glob.glob(f"{base}/{pat}")):
        fm = open(f, encoding="utf-8").read().split("---")[1]
        g = lambda k: (re.search(rf"^{k}:\s*(.*)$", fm, re.M) or [None, ""])[1]
        print(f"{'plugin' if base == plug else 'repo'} | {g('name')} | {g('model') or '-'} | {g('description')[:150]}")
EOF
```

Built-in skills and agents are already listed in the session context — do not search for them. `## Executors` in the profile (if present) adds extra folders and repo-specific restrictions.

A stage is covered by an existing tool if its description matches the stage's **action**, not just its topic. Partly covered → executor = that tool + a line "what is missing".

## 2. Model and effort

| Kind of work | Model (`model` in the call) | effort |
|---|---|---|
| Mechanical: moving files, bulk replacement from a ready table, `grep` inventory, link checks | `haiku` | low |
| Mechanical with small judgement: an edit from a clear pattern, turning reading into a table | `sonnet` | medium |
| Judgement: code with decisions, texts, migrations, root-cause analysis | `opus` | high |
| Planning a large task (`checkride:architect`), `checkride:validator` | the strongest model you have | high (set in both agents' frontmatter) |

An agent with `model:` in its frontmatter uses it; otherwise **pass the model explicitly in every call** — without it a subagent inherits the session's model, and a mechanical job on the most expensive model can cost hundreds of thousands of tokens.

## 3. Mode and brief

| Mode | When | How to launch |
|---|---|---|
| **inline** | a small stage (≲15 tool calls), needs the current context, the same files as the neighbouring stage | the main session |
| **subagent** | a cheaper model does the same job, **or** heavy reading (dozens of files, logs) the main session does not need to carry | `Task`/`Agent` with `model` and `effort`; result = a short report |
| **session in a worktree** | a long independent branch with its own files and its own result (a commit on its own branch) | a separate session with its own worktree, model and effort; brief as below |

An isolated run = **a fresh subagent with a brief**: a forked subagent carries the whole session context, and a nested `claude -p` launched from a session is unreliable.

Give a mechanical executor **a ready list in the brief** (files, a replacement table, the check command). It does not do recon: recon is what inflates tokens.

**A brief for a subagent or a session** (the executor starts without the session's context):
1. **Mission and reading** — one line on what, why and which stage; exact paths to read; from a shared file (the task file) only its own section: every extra re-read is multiplied by the number of agents.
2. **Spec and report** — the goal, hard constraints, the stage's check command; a report in numbers (counters, check output, artifact path), not a list of files. If the stage depends on a precondition, put the precondition and its check command in the brief: the executor runs it first; a mismatch → stop and report.
3. **Discipline** — write nothing outside its stage's files (in particular `CLAUDE.md`, auto-loaded rules, production) · absolute paths, no `cd` in compound commands · for a subagent: do not spawn your own subagents, do not use background runs · the brief is self-contained: if the executor dies, re-run the same brief unchanged.

**Long runs (ETA over 30–40 minutes)** — in the brief (commands below assume a Unix shell; adapt them on Windows): first check with `pgrep -f` that the process is not already running; launch it as a separate process (`nohup … & disown`) that survives turn boundaries and the agent's death, and return `launched` + PID + log path. Set up a background watcher that also catches crashes: the process is dead without a completion marker → report + log tail.

## 4. The graph and real parallelism

Stages A and B are parallel only if **all three** hold:
- their file sets do not overlap (including shared `CLAUDE.md`, registries, `settings.json`);
- they share no production system (the same service, database, workflow, schedule, channel);
- there is no deploy or migration order between them.

Otherwise run them in sequence ("after S1"). Parallel work that collides on a shared file is a merge conflict waiting to happen.

## 5. Tokens — when an agent pays off

- **Parallelism buys time, not tokens.** Each agent reads its own context: the system prompt, `CLAUDE.md`, the stage's files. Two agents at 40K each = 80K, not 40K.
- An agent is justified if (a) its model is cheaper, or (b) it isolates heavy reading that would otherwise ride along in every later turn of the main session.
- Small stages inline: starting an agent costs ~15–25K for context alone.
- Forecast an order of magnitude, not a precise number: `~10K` · `~50K` · `~200K`.

## 6. Writing it into the task file

The executor goes into the "Stages" table column: `inline` or `<agent or skill> · <model> · <mode> · ≈tokens`; dependencies and parallelism there too, only when the order is not sequential ("after S1", "in parallel with S2"). If there is at least one agent or session, add one line under the table: "Agents ≈N; pay off through <model / isolation>". Everything inline → write nothing.

## 7. Launch and acceptance

- Call a subagent with `model` and `effort` from the table and a brief per §3. Any deviation → a line in "Changes and cuts".
- A worktree session starts when all its predecessors have PASS. Its result is a branch, not the main branch.
- **Before merging** — the stage check on the branch (the command from the task file, output into the stage row). **Merge into the main branch only with the user's "yes"**, separately every time.
- **A subagent's or session's report is not an artifact** (proof table in `checks.md`): before PASS/FAIL, check the artifact itself; for a session, `git log`/`diff` of its branch before asking for a "yes". If an agent "delegated to the background" and came back without an artifact → tell it to do the work itself with its own tools (or re-run the same brief).
- **Tokens: actual vs forecast.** After each agent stage, take the counter from the call result into the stage row and the checkpoint; actual ≥3× the forecast and above 100K → stop before the next agent and give the user the cause (most often: the agent inherited the session's model) and the numbers. Inline stages have no counter — do not estimate by eye.
- **A requirement that appeared after agents started will not apply to artifacts already written:** send it to the active agents and compare the modification time of finished files with the time of the requirement; anything written earlier gets an explicit follow-up task.
- **A stopped session loses everything uncommitted in its worktree:** the brief tells a session to commit its results and research to its own branch right away; before removing a worktree, check `git -C <worktree> status --porcelain` and save anything uncommitted first.
