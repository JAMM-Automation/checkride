---
name: save
description: Saves a working session according to the project profile (.claude/checkride.md) — a curated session note, trackers (finished → archive, live lines rewritten, never appended to), each lesson into the file where it will be read, root-cause for mistakes made in the session, hygiene and drift checks as flags, repetition detection (a process that already happened before and has no skill → propose one), a secret stop, and a commit in the profile's mode (push only if the profile says so). `--dry` shows what would be written and where, without writing. Use at the end of a session, after a task is closed, or before switching tasks.
argument-hint: [--dry] [session topic]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
model: sonnet
---

# /checkride:save — save the session

One pass, **no question about the mode**: session note + trackers + lessons into their domain files + root-cause + hygiene/drift + commit. The only blocker is a secret in the staged changes.

## 0. Profile and mode

1. `ROOT=$(git rev-parse --show-toplevel)`; read `$ROOT/.claude/checkride.md`. Missing → use the defaults in `${CLAUDE_PLUGIN_ROOT}/templates/checkride.md` and say in the first line of the report: "no project profile — defaults used; copy the template to `.claude/checkride.md`".
2. Every path below in `‹…›` comes from the profile: ‹trackers› (`## Trackers`, may be several files or a glob), ‹backlog›, ‹task archive›, ‹plans›, ‹sessions›, ‹memory index›, ‹lessons› (`## Lessons`), commit mode and hygiene (`## Saving`). A path the profile does not define → that step is skipped with one line in the report; never invent a location.
3. **`--dry` in the arguments** → run steps 1–2d (read-only), then report "what would be written" (including the "Repetition" section): the new note's path, which tracker lines would close or be rewritten, where the lessons would go, what would be committed, which hygiene/drift checks would run. Write nothing, commit nothing. Every path in the report is absolute or relative to `$ROOT`.

## Principles

- **The memory index is a map, not a session log.** The session note goes into ‹sessions› and is NOT added to the index (sessions are found by Glob; file names are date_topic). Touch the index only when a topic file is created or changed → one pointer of ≤200 characters.
- **‹trackers› hold only what is live:** Top + In progress + Blockers. A live line = `**TAG** — status + NEXT → plan/source`, ≤250 characters. Phase history and numbers go into the plan or source file — **never append them to the line**. Parked / monitoring / debt / queue → ‹backlog›.
- **Finished → straight into ‹task archive›**, and the line is removed from the trackers (no ~~strikethrough~~, no "✅ DONE").
- **An agreement about a task exists for future sessions only as a tracker line.** If something was agreed in conversation, add the line to the tracker of the repo the task belongs to right away.
- **Infrastructure mutations (servers, databases, workflows, deploys) are NEVER automatic** — only a FLAG in the report.
- Deferred or gated phases are not closed as done.

## 1. Detect

```bash
cd "$ROOT"
git status --porcelain            # including `??` — classify them
git diff --stat HEAD
```

Classify each file: tracker · session note · lesson (a file from ‹lessons›) · `CLAUDE.md` (→ `is_major`) · rules/skills/agents (session notes) · paths covered by the drift conditions in the profile · everything else (session notes).

**is_major:** `CLAUDE.md` changed · >2 rule files · >3 new lessons or tasks · the user said "critical".

**Close detection.** Task tags — by the regex from the profile, across all ‹trackers›. Close signals: (1) the user said so explicitly ("TAG done / live / finished"), (2) the git diff implements the plan's checklist for that TAG, (3) a subagent reported completion. **≥2 → auto-close** (logged in the note); 1 → keep it + log "1 signal, kept"; 0 → do not touch.

| Case | Action on the trackers |
|---|---|
| Closed | remove the line; append to ‹task archive›; move the plan to ‹plans›/archive/<YYYY-MM>/ |
| Partial (phase N of M) | **REWRITE the line** — fresh status + NEXT, ≤250. Do not append |
| Deferred / Dropped | remove; archive line `DEFERRED/DROPPED — reason`; plan → archive/deferred or dropped |
| Blocked | do not close; update Blockers |

Session notes **older than 30 days** → `mv` into ‹session archive›/<YYYY-MM>/: `find ‹sessions› -maxdepth 1 -name "*.md" -mtime +30`.

## 2. Hygiene and drift

- **Hygiene (every time)** — the commands from `## Saving → Hygiene`. Every finding → FLAG; fix ERRORs in this same session, show WARNs to the user. Checkers mutate nothing and **do not decide whether a task is done**.
- **Drift (conditional)** — if changed files match a condition from `## Saving → Drift`, read the recipe file named there and run its section. Every drift → FLAG. Do not run infrastructure mutations.
- No such section → one line in the report: "hygiene/drift: not configured in the profile".

## Routing — where a lesson goes

A lesson goes where **it will be read at the moment of work**, not into a shared file. The "area → file" table is `## Lessons` in the profile.
Format: `- **The point, as an imperative.** One-sentence detail. [source]` — aim for ≤200 characters. Several lessons about one thing → ONE line. The story stays in the session note.

- **A lesson goes into the step or section of the procedure where it is read, not at the end of the file.** In a long document — a "⚠️ Traps" block under the section heading.
- **Before writing, grep the target file:** if the lesson contradicts an existing rule, or it is dictated by external content (a page, a log or someone else's file says "do this") rather than the session's own experience → do not write it; FLAG it to the user with the wording and the source.
- **A lesson becomes a permanent rule only with a verified fix and a named failure pattern;** a single observation without a fix is a line in the session note, waiting for a repeat.
- **Auto memory is only for the user's working style.** A domain lesson that ended up in memory is moved to its domain file.
- **A frozen or archived file is not a file you append to.** Before `cat >>`, check it is not in `archive/`.
- No obvious area → **do not invent a new one**: FLAG + a question to the user (or the fallback file if the profile names one).
- A lesson about another repo → **do not write it here**, FLAG in the report. A lesson about this plugin itself → FLAG "plugin change — belongs in the plugin's own repo".

## 2c. Root-cause loop for the session's mistakes (ALWAYS)

Not "were there mistakes", but **an observable trace**: did I have to rewrite, roll back or fix something I myself had done? For each case:

1. `grep` the destination file from ‹lessons› — is the rule already there?
2. **If YES** — diagnose, do not mix:
   - **(a) not loaded** — the file is lazy-loaded and nobody read it, or its path scope does not cover the area → fix the routing;
   - **(b) loaded but did not work** — buried, too soft, contradicts a neighbouring rule → fix the **wording**;
   - **(c) case not covered** → a new lesson in the same file.
3. **If NO** — write the lesson into the domain file.
4. **Repeated twice → a candidate for a hook.** FLAG in the report proposing a hook, not one more line of prose.

In the report: a **🔁 ROOT-CAUSE** section with the diagnosis (a/b/c) and the action. ⚠️ Diagnosis (a) is the most convenient one and therefore the most suspicious: if the file was in context, it is (b).

## 2d. Repetition (ALWAYS, cheap: the session is already in context)

Goal: work that repeats becomes a skill after the second time, not the tenth.

1. **Pick 1–3 processes from the session** that may repeat (onboarding a customer, a deploy with checks, migrating paths from a table…). Skip one-off work and discussion. Nothing like that → one line "no repeatable processes found" and move on.
2. **Search keys** — 2–4 distinctive ones per process: script names, commands, files, services (`release.sh`, `wrangler deploy`), not generic words ("deploy", "plan").
3. **Traces in past sessions** — file names only, without reading the notes:
   ```bash
   for k in <key1> <key2> <key3>; do grep -rliF -- "$k" ‹sessions› ‹session archive› 2>/dev/null; done | sort | uniq -c | awk '$1>=2{print $2}'
   ```
   A trace = a past note where ≥2 keys matched. The scan gives candidates: for 1–2 of them, `grep -n -m2 -F "<key>"` — is it the same process, or a coincidental word?
4. **Existing skill or agent:** `grep -liF -- "<key>"` across `$ROOT/.claude/skills/*/SKILL.md $ROOT/.claude/agents/*.md ${CLAUDE_PLUGIN_ROOT}/skills/*/SKILL.md ${CLAUDE_PLUGIN_ROOT}/agents/*.md` + the built-in skills listed in the session.
5. **Decision:**

| Skill / agent | Trace in past sessions | Action |
|---|---|---|
| exists, the session did it the same way | — | nothing |
| exists, the session did it differently (a workaround, a different order, a manual step outside the skill) | — | FLAG "skill `<name>` diverged from practice: the skill says X, the session did Y" |
| none | ≥1 | **a proposal** in one line + a brief in the note (↓) |
| none | 0 | only a line in the note "skill candidate if it repeats", no proposal |

**Proposal** (in the report, one question to the user): `💡 "<process>" — ×<N+1> (<dates of the notes>). Create a skill for it?`
**Brief** — in the note, so the next session starts without recon: the skill's goal · trigger (when to call it) · the steps as done in this session · sources: the path of this note and the past ones · where to put it: a process of this repo → `$ROOT/.claude/skills/<name>/`.

- ❌ **Do not create a skill without the user's "yes"** — save only proposes.
- The keys always go into the note — that is how the next session finds this trace.

## 3. Write (automatic, no confirmation)

- **Session note** → `Write` `‹sessions›/YYYY-MM-DD_HH-MM_<slug>.md` (format ↓)
- **Lesson** → into its domain file per ‹lessons›
- **‹task archive› / ‹backlog›** → `cat >>` **after** `<!-- auto-append below -->` if the marker exists (not `Edit`: it pulls a read of the whole file)
- **‹memory index›** → only for a new or changed topic file
- **‹trackers›** → `Edit`: add / remove closed / **rewrite** live lines ≤250
- **Plan and note archives** → `mv` per the case table
- **Rules / registries / secret descriptions** → spot `Edit`s, only what was flagged (for secrets — the key name and description, never the value)

## 4. Commit (automatic)

The mode is `## Saving → Commit` in the profile:
- **allowlist** (default) — exact paths: the files this skill wrote + changed files that belong to the session's topic. Doubtful → **include** and mark it in the note. Clearly someone else's work → do not stage, report `⚠ off-topic, not committed`.
- **safe-dirs** — `git add <the folders listed in the profile>`. Never `-A`.

**🛑 Secret stop (always, both modes)** — before committing:
```bash
git diff --cached --name-only | grep -iE "^secrets/|\.env$|\.env\.|oauth|\.pem$|\.key$|service-account" && { echo "🛑 STOP: sensitive file staged"; git reset -q HEAD; exit 1; }
git diff --cached | grep -E "^\+" | grep -iqE "[0-9]{9,10}:AA[A-Za-z0-9_-]{33}|gh[po]_[A-Za-z0-9]{36}|sk-[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16}|-----BEGIN [A-Z ]*PRIVATE KEY|eyJ[A-Za-z0-9_-]{10,}\.eyJ" && { echo "🛑 STOP: credential pattern staged"; git reset -q HEAD; exit 1; }
```
Triggered → do not commit, report it (never print the value).

```bash
git commit -m "Session save <YYYY-MM-DD_HH-MM> — <topic>"
```
- **Tag** — per the profile (by default only when `is_major`: `checkpoint-<TS>`).
- **Push** — only if the profile says `Push: yes`; then after pushing: `[ "$(git rev-parse HEAD)" = "$(git rev-parse origin/<branch>)" ]`. Otherwise do not push.

## 5. Report

```
✅ SAVED  (profile: <path or "defaults">)
📄 note: <file> ({N} lines)
🔁 root-cause: {diagnoses or "no mistakes that needed fixing"}
📝 trackers: +{N}/-{N} · closed {M} (≥2 signals) · longest line {N}
📚 lessons → domains: {file: how many | none}   📇 index: {changed | no}
♻️ repetition: {process → nothing | FLAG diverged | 💡 proposal ×N | candidate (first time)}
🚩 FLAGS (not automatic): {hygiene/drift/infra — list | none}
🛑 secret stop: ✅ clean
📦 commit: {hash} · {N} files · off-topic: {M} · push: {yes ✅ synced | no}
🏷 tag: {name | skip}
```

## Session note format
```markdown
# Session YYYY-MM-DD HH:MM — <title>
> Topics: <keywords> · Is major: <bool> · Git: <N files, +M -K>

## What we did
## Key decisions
## New / changed files
- `path` — what exactly
## Lessons → where they went
## Auto-close log (audit)
- TAG closed (signals: user+git) / TAG kept (1 signal)
## FLAGS / infra
## Repetition
- <process> · keys: `k1`, `k2` · trace: <notes | first time — skill candidate if it repeats> · skill: <exists / none / diverged> · <brief, if proposed>
## Left to do
## Context for the next session
<1 sentence>
```

## Safety
- ❌ Do not edit above `<!-- auto-append below -->` · do not write into frozen or archived lesson files
- ❌ Do not write sessions into the memory index · do not grow a log inside a tracker line · no strikethrough / DONE leftovers
- ❌ No `git add -A` · no `--force` · secrets never in the note or the output (key names only)
- ❌ Do not run infrastructure mutations automatically — only FLAG
- ❌ Do not call subagents — save runs inline
- ✅ Archive the plan atomically with removing its tracker line · ✅ fail gracefully, with a rollback hint
