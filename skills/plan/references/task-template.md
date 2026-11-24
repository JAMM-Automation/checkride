# Task file — template for `/checkride:plan`

> One file per task: plan and state together. Stages and criteria are written once, with status and proof in the same table; there is no separate log.
> **Rule:** the file is the truth of the task; the session context is a cache. Re-read it before every stage, update it after every stage.
> **No filler:** a medium task ≈30 lines, a large one ≈60. Bulky material (exact text edits, SQL, specs) goes into attachments in `‹plans›/<slug>/`; here only a link.

Path: `‹plans from the profile›/<slug>.md`.

---

```markdown
# <task name>
> Size: medium or large · Status: plan / execution / closed · Created: YYYY-MM-DD

## Goal and requirements
> What must become true and what the user called mandatory. Not rewritten after approval: changes go into "Changes and cuts".
- <1–5 lines>

## The user's decisions
> Not reworded after the fact.
- <question> → **<answer>** (date)

## Stages
| # | Stage | Executor · model · mode | Check: PASS / typical FAIL | Status · proof |
|---|-------|-------------------------|----------------------------|----------------|
| 1 | | inline | | ⬜ |

## Done when
| # | Criterion — a fact a command can refute | Command → expected | Status · proof |
|---|---|---|---|
| D1 | | | ⬜ |

## Rollback
- <one or two actions; for irreversible steps — what exactly cannot be rolled back>

## Changes and cuts
- YYYY-MM-DD · <what changed, was rejected, or turned after a FAIL: symptom → cause> · reason · effect on work already done

## Open
- [ ] <date · what is pending: a temporary file or record, a disabled guard, "handed over" from an agent report (from whom · what · to whom), a question>
```

---

## How to use it

- **Created at the "Plan" step, before approval;** requirements and the user's decisions go in right away.
- **Statuses:** ⬜ not started · 🔄 in progress · ✅ closed with proof · ❌ failed · ⚠️ not machine-checkable. Proof is command output or a commit hash, not a retelling.
- **After approval, the content of stage and criteria rows is not rewritten** — only "Status · proof" changes; a content change is a line in "Changes and cuts". For a large task, the copy at approval time lives in `<slug>/approved.md`. Rewriting a row is how requirements silently disappear.
- **Removing a requirement or a criterion** is allowed only as a line in "Changes and cuts" — that is how the validator tells a deliberate decision from a lost requirement.
- **"Open" is empty before close:** every line is closed or moved to the backlog.
- **What does not belong here:** a journal (history is `git log` plus the proof in stage rows), lessons (`/checkride:save` puts them where they will be read; here only a one-line turn after a FAIL), retold context, criteria written twice.
- **Resuming in a new session:** read the file → the first row in "Stages" that is not ✅. That is why statuses must be honest and updated after every stage, not at the end.
