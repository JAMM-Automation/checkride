# checkride

**A Claude Code workflow where nothing is called done without an independent check.**

In aviation, a checkride is the flight test where an examiner who did not fly with you decides whether you are cleared. `checkride` brings the same idea to agent work: plan the task, let an independent skeptic challenge the plan, execute it stage by stage with a check after each stage, and close it only when every done-criterion is proven by a command — checked by a validator that never reads the executor's report.

```
recon ─▶ plan ─▶ skeptic ─▶ your approval ─▶ stage ▸ check ▸ stage ▸ check ─▶ criteria proven ─▶ validator ─▶ save
                 (clean context,                (proof is command output,       (reads the live system,
                  no author reasoning)           not a summary)                   not the report)
```

## Why

Agents are fast and confident. The failures that cost the most are not syntax errors — they are:

- a plan that was more complex than needed, because nobody pushed back;
- "done" reported for work that was not actually checked;
- a requirement that quietly disappeared when a plan row was rewritten;
- a "disabled" path that still works through a second route nobody looked for;
- a mistake that repeats next week because the lesson went into a chat, not into the file that is read at the moment of work.

`checkride` is a small set of skills and agents built around those failures. It is plain Markdown — no servers, no dependencies — and it adapts to any repo through one profile file.

## What is inside

| Component | Type | What it does |
|---|---|---|
| `/checkride:plan` | skill | The single entry point for any task bigger than one edit. Sizes the task, does recon, asks one round of questions (each with a recommendation), writes one task file with stages, checks and done-criteria, waits for your approval, executes stage by stage, closes only with proof. |
| `/checkride:grill` | skill | A structured interrogation before work: a decision tree asked in rounds until nothing is silently assumed. Ends with a shared understanding and machine-checkable done-criteria. |
| `/checkride:save` | skill | Saves the session: a curated note, tracker updates, each lesson into the file where it will be read, a root-cause pass on the session's own mistakes, detection of processes that should become skills, a secret stop, a commit. |
| `checkride:skeptic` | agent | The "tenth man". Gets the finished plan without the author's reasoning and challenges it through three lenses: simpler? fragile assumptions? existing reuse? Returns STOP / CONCERNS / PASS. |
| `checkride:architect` | agent | Plans large changes when there is a real design fork: alternatives, target state, risks, migration order, criteria. Read-only; returns the task file text. |
| `checkride:validator` | agent | Checks the live system before a large task is closed. Read-only, never sees the executor's report, and also hunts for what could have been missed: lost requirements, consequences of mid-task changes, references to cancelled things, forgotten traces. |
| `checkride:creative-director` | agent | Improves the delivery of a finished draft (hook, one benefit, one action, rhythm) without touching facts, links or placeholders. Scores its own output and rewrites anything below 7/10. |
| `templates/CLAUDE.md` | template | The Plan → Act → Verify rules to paste into your project's `CLAUDE.md`. |
| `templates/checkride.md` | template | The project profile: where your trackers, plans, lessons and registries live, what "production" means, how to commit. |

## Install

In Claude Code:

```
/plugin marketplace add JAMM-Automation/checkride
/plugin install checkride@jamm-automation
```

Then, optionally:

1. Copy `templates/checkride.md` to `.claude/checkride.md` in your repo and fill in what applies. Without it, the skills use sensible defaults (`.claude/plans/`, `.claude/sessions/`, `TODO.md`) and say so.
2. Paste `templates/CLAUDE.md` into your project's `CLAUDE.md`, or reference it from there.
3. If you want Claude to call the skills on its own without a permission prompt, allow them in `.claude/settings.json`:
   ```json
   { "permissions": { "allow": ["Skill(checkride:*)"] } }
   ```

To try it without installing: `claude --plugin-dir /path/to/checkride`.

## Quick start

```
/checkride:plan Add rate limiting to the public API: 100 requests per minute per key, 429 with Retry-After
```

What happens:

1. **Size.** This touches production → large.
2. **Recon.** Claude reads the router, the middleware and the existing limits itself, then asks only what it cannot read — each question with a recommended answer.
3. **Plan.** One task file in `.claude/plans/` with stages (each with a PASS value and the typical FAIL values written before the run), 3–7 done-criteria that a command can refute, and a rollback.
4. **Skeptic.** `checkride:skeptic` reads the file in a clean context and returns its verdict; concerns are applied before you see the plan.
5. **Approval.** You get "before → after", the stages and the criteria. Nothing has changed except the task file until you say go.
6. **Execution.** Stage by stage, with a one-line checkpoint after each: `Stage 2/4 — middleware. PASS: 101st request → 429, Retry-After: 37. Next: per-key counters.`
7. **Close.** Every criterion with its command and output; `checkride:validator` checks the live API on its own; then `/checkride:save`.

A full walkthrough is in [`examples/walkthrough.md`](examples/walkthrough.md).

## How much process

Depth scales with risk. A one-line fix gets three lines in chat (what · how I check · the proof) and no task file. A medium change gets a task file and your approval. Only large, risky work gets the skeptic, the architect and the validator. If the process costs more than the risk it removes, the skills are written to cut it.

## Principles it enforces

- **The task file is the state; the session is a cache.** It is re-read before every stage, so long sessions do not drift.
- **Criteria are written before the work,** as facts a command can refute — never "it works".
- **Check both directions:** "the new path works" and "the old path is really gone".
- **An agent's report is not an artifact.** Check the file, the row, the response.
- **A negative test without a positive control proves nothing.**
- **Irreversible steps get their own "yes",** even inside an approved plan.
- **A lesson goes where it will be read,** not into a shared notes file.

## Compatibility

Written for Claude Code. The skills are plain Markdown and the ideas are tool-agnostic, but other agent tools have not been tried and are not claimed as supported.

The workflow reduces the chance of an unchecked "done"; it does not guarantee correct results. Provided as is — see [LICENSE](LICENSE).

## Credits

`/checkride:grill` is adapted from the grilling approach and the `grill-me` skill by Matt Pocock ([mattpocock/skills](https://github.com/mattpocock/skills), MIT). Full notice: [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md).

## License

[MIT](LICENSE)

---

Built by [JAMM Automation](https://github.com/JAMM-Automation).
