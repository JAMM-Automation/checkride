# Done-criteria and PASS/FAIL — how to write checks that catch things

> Read at the "Plan" step and before checking the criteria at close. You do not need to keep it in your head — this is the cheat sheet.

## Two different levels — do not mix them

| | **Done-criterion** | **Stage check (PASS/FAIL)** |
|---|---|---|
| About | the task as a whole | one step of work |
| Written | BEFORE execution, in the plan | BEFORE execution, in the plan |
| Checked by | me at close; for a large task that changed production or did something irreversible — also `checkride:validator` | me, right after the stage |
| Wording | "what must be true in the world" | "which command and which output" |

## How much checking is enough

- **Proportional to risk.** Text and config in git — a diff and one command per criterion. Write a checker only when the same check repeats after every stage or guards against a regression in production.
- **3–7 criteria per task.** More is a sign the criteria repeat the stage checks.

## How to word a done-criterion

**Bad:** "the referral program works" · "the page is done" · "the cron is fixed"
**Good:** a fact that can be refuted:
- "a visit through someone else's link + a payment → both accounts' `paid_until` moved by 30 days"
- "a repeated visit by the same user does not create a second credit"
- "limit 3: the fourth referral earns no bonus"

The fitness test: **can I imagine a command that would refute it?** If not, the criterion is not operational — rewrite it.

## PASS/FAIL format

Write it **together with the expected output**, before the run. Otherwise it is easy to "draw" the expectation around whatever came out.

```bash
# A. Each lookup runs EXACTLY once, the charge runs exactly twice
# PASS: every "Lookup — …": 1   ·   "Charge": 2
# FAIL: Charge == 8   → the loop is not fixed, STOP, do not enable real charging
# FAIL: Charge == 1   → the loop-back is broken, STOP
# FAIL: no "Charge" key → the step never ran at all
```

Three parts, all mandatory:
1. **What we check**, one line
2. **PASS:** the exact value
3. **FAIL:** not just "otherwise", but **a breakdown of typical bad values** — each one points to its own defect

The last part is the most valuable: `== 8` and `== 1` break for different reasons, and knowing which saves an hour of diagnosis.

## Check both directions

The most expensive class of mistake: proving that a branch is **alive** without proving that another branch is **dead**. It often is not (a second path in the graph, old buttons or links that users still hold).

For any "disabled / removed / no longer available":
- ❌ not enough: the direct path does not work
- ✅ needed: **no workaround path exists** — walk the graph, all entry points, old artifacts in users' hands

## What is proof and what is not

| Not proof | Proof |
|---|---|
| "changed the config" | the runtime responds the new way |
| "the step is in the workflow" | the step is **reachable** — connections checked |
| "the handler is written" | the bot actually answered the command |
| the documentation says | the database / API / HTTP status says |
| an agent's report | `ls` / `wc` / `grep` of the artifact |

- **Sync docs against the TEXT of the criterion, not memory:** if a criterion names specific sections, grep every named section.
- **A grep for an identifier in a criterion is case-sensitive and bounded:** `order_id` also matches `order_id_hash`, so the criterion never reaches 0. Use `order_id([^_a-z]|$)`.

## Traps in grep and done-checkers

> Read before every new checker or assertion (the "Execution" step), not only at planning and close.

- **A grep for forbidden words catches substrings:** `USD` fires on `USDT`. Put word boundaries on both sides (`\bUSD\b`, `grep -w`) and check the neighbouring word in the same run. Do not quote file paths inside the texts being checked; the proof in the criteria table must not contain the very string the check searches for.
- **Counting removed diff lines:** `grep -E '^-' | grep -vc '^---'`, not `grep -cE '^-[^-]'` — changed markdown list items look like `-- …` in a diff and the second regex misses them.
- **A criterion-grep "over the whole file" trips on the plan's own service text** — and on notes the session adds later (header, history line). Check only marked public blocks (`<!-- copy:start -->` … ) or lines with a fixed prefix; write session notes without the terms the checker counts.
- **A boundary marker in a checker is anchored at line start, not a substring:** splitting on the first `✂` cuts at a `✂` already present in the header — a false FAIL on correct text. Use `re.search(r'(?m)^✂')`, plus a positive control with the marker inside the text.
- **After every stage, run the whole checker, not only this stage's criteria.** FAILs of future stages are expected and listed in the task file; a new FAIL outside that list is a regression.
- **No `|` inside cells of a markdown table that a checker parses** — not even `\|`, not in your own notes either: `split('|')` shifts the columns. After every edit, check the number of `|` in every criteria row.
- **A positive control must be plausible in size, not just "> 0":** a pattern that matches 117 files when you expected 4 is broken (a one-letter fragment from a regex example). Compare "before" with the expected list before trusting "0 after"; do not use patterns shorter than 6 characters.
- **Accumulate state across languages or files; do not overwrite it:** `flag = "ok" if good else "MISSING"` in a loop lets a later "ok" erase an earlier "MISSING". Write `flag = "MISSING" if (flag == "MISSING" or not good) else "ok"`; run the self-test on every input, not just the first.
- **Extracting visible text from HTML — no spaces between inline elements:** "n<span>8</span>n" becomes "n 8 n" and causes false "text missing". Insert separators only at block tags; join inline content without spaces.
- **A checker that reads `git status` gives a false FAIL on an uncommitted tree.** Run baseline and final passes of such checkers only on a clean tree (`git status --porcelain` empty), and say so in the criterion.
- **A spot fix to someone else's checker:** first find out what the line reads — `git diff <range>` (historical expectations stay historical) or the working tree (current ones). Replace "exactly N external links" with set equality against an allowlist; anchor a negative assertion "the old phrase is gone" on the form of the old rule, because the new text may legitimately use the same words.
- **A checker built on `git ls-files` does not see new untracked files:** run it after `git add` (or add `--others --exclude-standard`), otherwise "0 findings" before the commit is a lie.

## `NOT-VERIFIABLE` — a legal status

If a criterion cannot be checked without mutating production or without access you do not have, mark it explicitly and hand it to the user for a manual check. **Silently skipping it is not allowed:** an unmarked item reads as passed.

Many `NOT-VERIFIABLE` items are a signal that the criteria were written abstractly. Normal is 0–1 per task.
