---
checkride-profile: 1
---
# checkride project profile

> Copy this file to `.claude/checkride.md` in your repo and edit it. Every section is optional: a missing section means "not used in this project" and the step that reads it is skipped with one line in the report.
> When no profile exists at all, the skills use the defaults written below as-is. Lines in `<angle brackets>` are placeholders: they are ignored until you replace them with real values.
> Skills and agents never contain repo paths of their own — everything project-specific lives here.

## Project
- Name: <repo name> · main branch: `main`
- Report language: the language the user writes in
- Secrets: <where they live and how to read them, e.g. "environment variables only, never print values">

## Guards
> Work that does NOT belong in this repo. Format: signs of the task → the redirect sentence. The skills stop after the sentence.
- <e.g. "Mobile app, iOS, Android → «This belongs to the mobile repo — open a session there.»">

## Trackers
- Priorities: `TODO.md` (sections: Top / In progress / Blockers; one line per task, ≤250 characters)
- Backlog: `.claude/backlog.md`
- Task archive: `.claude/archive-tasks.md` (append after `<!-- auto-append below -->`)
- Plans: `.claude/plans/` · archive `.claude/plans/archive/<YYYY-MM>/`
- Sessions: `.claude/sessions/` · archive `.claude/sessions/archive/<YYYY-MM>/` (after 30 days)
- Memory index: none
- Task tag regex: `\*\*([A-Z0-9-]{3,})\*\*`

## Lessons
> Where a lesson goes, so it is read at the moment of work. Area → file → when it gets loaded.

| Area | File | Loaded |
|---|---|---|
| everything else (fallback) | `.claude/lessons.md` | read by `/checkride:plan` during recon |
| <e.g. database> | <`docs/db.md`> | <when the task touches the database> |

## Registries
> Where to check what already exists before proposing a new tool (the skeptic's reuse lens, the plan's recon).
- `.claude/skills/`, `.claude/agents/`, `scripts/`
- <e.g. `docs/services.md` — every deployed service and its owner>

## Executors
> Optional: extra skill/agent folders, forbidden modes, fan-out limits, long-run specifics.
- Parallel subagents: at most 5 without asking
- <e.g. "a worktree session never deploys">

## Production
> What "production" means here and how to read it without changing it. The validator reads only this section for access.
- <e.g. "API: https://api.example.com — read with `curl -s`; database: read-only role via `psql \"$DATABASE_URL_RO\"`">
- Forbidden for the validator: <anything beyond the default read-only rules>

## Saving
- Commit: allowlist (only files the session touched for its topic)
- Push: no
- Tag: only for major sessions
- Hygiene: <commands to run on every save, e.g. `npm run lint --silent`>
- Drift: <condition → recipe file, e.g. "migrations/ changed → read docs/db.md §Drift">

## Grilling
> Domain branches `/checkride:grill` must cover in addition to the shared ones.
- <e.g. "Pricing: does the change touch existing customers?">
- After the interrogation: <where to hand repo-specific work, e.g. "copy → the creative-director agent">

## Copy
> Read by the creative-director agent before editing any public text.
- Style guide: <path or "none">
- Forbidden phrases: <path or list>
- Legal rules for public text: <path or "none">
