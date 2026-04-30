# Heartbeats — Poof's built-in cron primitive

Poof has a first-class **Heartbeat** primitive for recurring backend jobs. **Use this for any digest emails, mark jobs, scheduled publishes, leaderboard recompute, summary rollups, mid-week alerts, or any other periodic work.** Do NOT spec or build:

- A local Bun / Node service with `cron` or `setInterval`
- A separate Cloudflare Worker with a `scheduled()` handler
- A GitHub Actions schedule that hits an API route
- A VPS cron job
- A Poofy-daemon side-channel scheduler

All of those are wrong for Poof products. Heartbeats run on the user worker (same project, same billing as HTTP requests) via the platform's dispatch namespace. Setting one up is a single `poof iterate` call.

## How it works

Poof uses a **dispatcher + dispatch namespace** architecture:

1. **Your worker** exports task handlers and registers heartbeat routes.
2. **`partyserver/heartbeat.json`** declares which tasks exist, their cron expressions, and enabled state.
3. At deploy time, the platform registers your schedules in a central KV registry.
4. A **heartbeat dispatcher worker** runs every minute, checks which crons match, and calls your worker's `/__internal/heartbeat/:taskName` route via the dispatch namespace binding.
5. The `/__internal/*` path is **blocked at the proxy** for public requests — only the dispatcher can reach it (no API keys or shared secrets needed).

Logs appear in the Backend Logs tab prefixed with `[heartbeat]`. Each invocation gets up to 300,000ms CPU time.

## Source-tree layout (the Poof AI generates these on iterate)

```
partyserver/
├── heartbeat.json                  # task config (name, cron, handler, enabled, schedule.live)
└── src/
    └── heartbeat/
        ├── index.ts                # registry + __internal/heartbeat/:taskName routes
        ├── weekly-publish.ts       # one file per task
        └── mark-outcomes.ts
```

`partyserver/src/index.ts` calls `registerHeartbeatRoutes(app)` to wire the routes. New projects include this scaffolding automatically.

## Adding a heartbeat task — agent recipe

Tell the Poof AI in `poof iterate` exactly what you want, including the cron, the handler logic, and any `schedule.live` overrides:

```bash
poof iterate -p <project-id> -m "Add a heartbeat task \`weekly-publish\` that runs every Monday at 13:30 UTC (cron: \`30 13 * * MON\`). Handler logic: run the two-AI orchestration pipeline (Stage 1–5 in spec.md) using OPENAI_API_KEY + ANTHROPIC_API_KEY from secrets, then atomically write the resulting basket via setMany — one weeks/\$weekId, 5–15 weeks/\$weekId/picks/\$pickId, 5–15 weeks/\$weekId/picks/\$pickId/private/r1, and one weeks/\$weekId/portfolio/r1. After successful write, dispatch the Monday digest email via AGENTMAIL_API_KEY from secrets. Wire it via partyserver/heartbeat.json + partyserver/src/heartbeat/weekly-publish.ts and register in partyserver/src/heartbeat/index.ts. Use schedule.live to keep the same cron in production."
```

Pattern: declarative description of what should run + when, not the source code. The Poof AI emits the handler, the JSON config, and the registry wiring.

## Per-environment scheduling — the recommended pattern

Top-level fields apply to **draft + preview** (development / staging). `schedule.live` overrides apply to **production**. Each field falls back to its top-level default if not set.

**The recommended pattern is to disable the schedule on draft + preview entirely** so dev iterations don't burn cron fires, then fire the same handler on demand via the manual-trigger path while you're working on it. Production runs on its real cron expression:

```json
{
  "tasks": [
    {
      "name": "weekly-publish",
      "cron": "30 13 * * MON",
      "handler": "./src/heartbeat/weekly-publish.ts",
      "enabled": false,
      "description": "Two-AI orchestration pipeline. Runs on prod schedule; manually triggered on draft.",
      "schedule": {
        "live": {
          "cron": "30 13 * * MON",
          "enabled": true
        }
      }
    }
  ]
}
```

Manual triggers (Heartbeat UI Run button or `poof chat`) ignore the `enabled` flag, so the task is fully runnable on draft for testing — it just doesn't fire on cron there.

When you genuinely need the cron to fire on draft + preview (e.g., a fast-iterating mark job during active development), set `enabled: true` at the top level. This is the exception, not the default.

## Manual trigger — the canonical "seed draft" path

When a UI surface depends on rows that the heartbeat task is supposed to publish (e.g., the inaugural week's basket), **don't write a separate seed script or `bootstrap-seed-data.json` lifecycle file**. Manually trigger the same heartbeat task on draft:

```bash
poof iterate -p <project-id> -m "Trigger the \`weekly-publish\` heartbeat task on draft now to populate the inaugural week. The task should write its rows using the publisher signer — same code path that runs on the prod schedule."
```

The Poof AI runs the task once via the dispatcher's manual-trigger path. Same handler code, same publisher signer, same writes — just out of schedule and on draft.

Why this is better than a parallel seed file:

- **Same code path.** The seed and the production schedule run the exact same handler. No drift between "seed" and "real" data.
- **Realistic data.** The handler does what the spec says it does — calls the real AI pipeline, writes the real shape of rows. Stitch-mockup placeholders never enter source.
- **No extra infra.** Bootstrap lifecycle files run in mock/test envs and don't write to the deployed draft DB. Manual heartbeat triggers do.
- **Doesn't burn the cron schedule.** Manual trigger is independent of cron — fire it as many times as you need during dev, prod cron is unaffected.

The Heartbeat tab in the platform UI also has a **Run** button that triggers the same path. The CLI / chat way is preferred for agents because it's scriptable and leaves a record in the project's chat history.

## Cron expression cheat sheet

| Expression | Meaning |
|-----------|---------|
| `*/5 * * * *` | Every 5 minutes |
| `*/15 13-22 * * 1-5` | Every 15 min, 13:00–22:59 UTC, Mon–Fri (US market hours) |
| `0 * * * *` | Every hour on the hour |
| `30 13 * * MON` | Mondays at 13:30 UTC |
| `0 0 * * *` | Daily at midnight UTC |
| `0 0 1 * *` | Monthly on the 1st at midnight UTC |

UTC only — no timezone field. Convert local times yourself.

## Important rules (cribbed from the in-project skill)

1. **Always log** task start/end and key metrics — `console.log("[<task>] …")` shows in the Backend Logs tab.
2. **Handle errors gracefully** — a failed task should not crash other tasks.
3. **One task = one responsibility.** Separate `weekly-publish` from `mark-outcomes`.
4. **Use `Promise.allSettled`** for fan-out so one failure doesn't kill the batch.
5. **Don't import API route handlers** — heartbeat tasks are independent.
6. **Don't call your own API routes from within a task** — direct DB / env access is available.
7. **Don't create test/trigger API routes** — the Heartbeat UI Run button (and the `poof chat`/`iterate` manual-trigger path) handle this. No `/api/test-heartbeat` or `/api/run-publish` routes needed.

## Spec convention (Poofy products)

When writing `spec.md` for a Poof product that has scheduled work, list it under `**Heartbeat tasks:**` with one row per task:

```markdown
## Architecture

**Poof tier:** Tier-3 (Frontend + Policy + Backend).

**Heartbeat tasks** (live in `partyserver/src/heartbeat/`, configured in `partyserver/heartbeat.json`):

- `weekly-publish` — `30 13 * * MON` (Mondays 13:30 UTC) — runs the two-AI orchestration pipeline, writes the week's basket via `setMany`, dispatches the Monday digest email.
- `mark-outcomes` — `*/15 13-22 * * 1-5` (every 15 min, US market hours) — polls market data, writes `outcomes/$weekId/$pickId` when target/stop hits.
```

Do not include a `**Backend service:**` section listing a Bun + TypeScript process or `bun run backend/src/jobs/...` ACs. There is no separate backend service — the heartbeat tasks ARE the backend job runtime.

For Stripe webhooks and other inbound HTTP integrations, use a regular **PartyServer route** (e.g., `POST /api/stripe/webhook`) — also in the same project, also no extra infra.
