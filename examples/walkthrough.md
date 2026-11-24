# Walkthrough — one task from plan to close

A fictional but realistic task in a small web service, showing what each step produces. Names, numbers and outputs are illustrative — this is not a record of a real run.

**The request:**

```
/checkride:plan Add rate limiting to the public API: 100 requests per minute per API key, 429 with Retry-After. Internal health checks must not be limited.
```

## 1. Size and recon

Claude decides **large** — it changes production behaviour for every client. Then it reads, without asking:

- the router and middleware chain (`src/server.ts`, `src/middleware/`);
- how API keys are resolved (`src/auth/key.ts`);
- whether a cache is already available (Redis is in `docker-compose.yml` and used for sessions);
- the profile `.claude/checkride.md` → plans folder, what "production" means, how to read it.

One round of questions — only what the code cannot answer:

```
❓ Q1 — Window type: fixed 60-second windows or a sliding window?
➡️ Sliding window. A fixed window lets a client send 200 requests across a window boundary; the sliding version costs one more Redis call.

❓ Q2 — Clients over the limit today: the logs show 3 keys above 100/min at peak. Notify them before enabling?
➡️ Yes: run the limiter in log-only mode for a day first, send the list to you, then enforce.
```

## 2. The task file

`.claude/plans/api-rate-limit.md` (shortened):

```markdown
# API rate limiting
> Size: large · Status: plan · Created: 2026-11-24

## Goal and requirements
- 100 req/min per API key, sliding window; over the limit → 429 + Retry-After
- /health and internal callers are never limited
- Log-only mode first, enforce after the user's go

## Stages
| # | Stage | Executor | Check: PASS / typical FAIL | Status |
|---|-------|----------|----------------------------|--------|
| 1 | Limiter middleware + Redis sliding window, behind LIMIT_MODE=off/log/enforce | inline | unit tests: 100 pass, 101st → limited / 101st passes → window key wrong | ⬜ |
| 2 | Exempt /health and internal network | inline | test: 1000 × /health → all 200 / any 429 → exemption order wrong | ⬜ |
| 3 | Deploy with LIMIT_MODE=log | inline · 🔴 separate yes | logs show "would-limit" lines, 0 × 429 served / 429 in access log → mode flag ignored | ⬜ |
| 4 | Switch to enforce | inline · 🔴 separate yes | 101st request from a test key → 429, Retry-After ≤ 60 | ⬜ |

## Done when
| # | Criterion | Command → expected | Status |
|---|---|---|---|
| D1 | 101st request in 60 s from one key is refused | `scripts/burst.sh KEY 101` → last status 429, Retry-After present | ⬜ |
| D2 | Health checks are never limited | `scripts/burst.sh --path /health 1000` → 1000 × 200 | ⬜ |
| D3 | Another key is unaffected by a limited key | burst key A to 101, then 1 request with key B → 200 | ⬜ |
| D4 | Turning it off restores old behaviour | LIMIT_MODE=off → 150 requests → 150 × 200 | ⬜ |

## Rollback
- LIMIT_MODE=off (no deploy needed)
```

## 3. The skeptic

`checkride:skeptic` gets only the file path:

```
## Verdict: CONCERNS
**The shortest path I see:** the plan is right-sized; one gap.
**Critical:**
- Stage 3 has no criterion for what the log-only day produces → add "list of keys over the limit delivered to the user" before stage 4.
**Hidden assumptions:**
- Redis is treated as always available → what happens to requests if Redis is down? Fail open or closed must be decided.
**Recommendation:** go-after-fix.
```

Both points go into the file. The second becomes a question at approval: *fail open (recommended — an outage of the limiter should not become an outage of the API)*.

## 4. Approval

You see "before → after", the four stages, the four criteria, the skeptic's verdict in one line and the fail-open question. You answer "go, fail open". Only now does any code change. The file is copied to `api-rate-limit/approved.md`.

## 5. Execution with checkpoints

```
Stage 1/4 — limiter. PASS: 100 allowed, 101st limited, window slides. Next: exemptions.
Stage 2/4 — exemptions. PASS: 1000 × /health → 200. Next: deploy in log-only mode (needs your yes).
```

Stage 3 is a deploy, so it asks for its own "yes" even though the plan was approved. After a day:

```
Stage 3/4 — log-only. PASS: 0 × 429 served, 3 keys would have been limited (list attached). Next: enforce (needs your yes).
```

## 6. Close

```
Planned 4 criteria, closed 4:
✅ D1 — scripts/burst.sh test-key 101 → 429, Retry-After: 41
✅ D2 — scripts/burst.sh --path /health 1000 → 1000 × 200
✅ D3 — key A limited; key B → 200
✅ D4 — LIMIT_MODE=off → 150 × 200, then back to enforce
```

Then `checkride:validator` — it gets the criteria and the file paths, not this summary. It runs its own requests against the live API, and its "what could be missed" pass finds one thing: the API docs page still says "no rate limits". That becomes a fix and a line in the task file.

You answer "close". `/checkride:save` writes the session note, moves the task line to the archive, puts the lesson "decide fail-open vs fail-closed for every new dependency in the request path" into the file your project reads for middleware work, and commits.
