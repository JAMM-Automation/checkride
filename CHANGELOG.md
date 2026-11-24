# Changelog

## 0.1.1 — 2026-11-24

- The architect agent gets the absolute paths of the plan references from the caller.
- `save` creates missing tracker and archive files (with the append marker) and archive folders.
- Placeholders in `<angle brackets>` in the profile template are ignored when no profile exists.
- `grill` blocks subagent fan-out under both tool names (`Task`, `Agent`).
- `creative-director` made surface-neutral: no chat-bot or single-button assumptions; marked optional.
- Examples in the agents made domain-neutral; Unix assumption noted for long runs.

## 0.1.0 — 2026-11-24

First public release.

- Skills: `plan`, `grill`, `save`.
- Agents: `architect`, `skeptic`, `validator`, `creative-director`.
- Templates: project profile (`templates/checkride.md`) with working defaults, `CLAUDE.md` rules.
- Example: a full walkthrough of one task from plan to close.
