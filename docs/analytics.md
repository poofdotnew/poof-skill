# Client App Analytics

Poof client apps are instrumented automatically at the Cloudflare edge. All current and future
`*.poof.new` apps served through Poof's Cloudflare Worker get first-party analytics without a
generated app rebuild or local frontend code changes.

## What Is Captured

Poof uses Cloudflare Workers Analytics Engine as the only analytics provider for client app
telemetry. The proxy injects a lightweight collector into HTML responses and records edge events
directly from the Worker.

Tracked traffic and routing events:

- `page_view`, `route_view`, `asset_request`, `api_request`
- `spa_fallback`, `r2_miss`
- anonymous visitor and session identifiers
- route buckets, referrer domain, UTM fields, country, colo, browser family, and device class

Tracked client failures:

- `js_error` from browser runtime errors
- `unhandled_rejection`
- `resource_error` for failed scripts, stylesheets, images, fonts, and media
- `api_error` for failed or 4xx/5xx `fetch` / `XMLHttpRequest` calls
- observable navigation failures

Tracked edge failures:

- `worker_exception`, `r2_miss`, `r2_read_error`, `dispatch_error`
- `blocked_internal_route`
- `static_4xx`, `static_5xx`, `api_4xx`, `api_5xx`

Tracked RUM/performance fields include TTFB, FCP, LCP, INP, CLS, navigation duration, resource timing
summaries, engaged seconds, and bounce/session estimates where the browser can report them.

## Privacy Rules

Analytics are intentionally coarse and operational:

- Do not capture wallet addresses, emails, IP addresses, auth tokens, request bodies, response bodies,
  or full stack traces by default.
- JS errors store error type/name, a hashed/normalized message, file origin/category, safe line/column
  data, route bucket, and release/build id when available.
- API and resource failures store method, same-origin/external class, normalized path bucket, status,
  duration, initiator type, and failure class.
- Worker failures store app/site attribution, route kind, status, exception class, operation name, colo,
  and duration.

Use this data for health, debugging, and product analytics. Do not treat it as a raw event log or a
source of user content.

## CLI Retrieval

Use `poof analytics` for deployed client app traffic, browser errors, failed resource/API loads, RUM,
and edge failure metrics. The command requires normal project access and the same paid-feature unlock
as other analytics/diagnostic surfaces: the wallet must have completed at least one credit purchase.

```bash
# Last 24h for draft/development
poof analytics -p <project-id>

# Preview traffic from the last hour
poof analytics -p <project-id> --environment preview --range 1h

# Production/live, with larger top-page/error lists
poof analytics -p <project-id> --environment production --range 7d --limit 25

# Machine-readable output
poof analytics -p <project-id> --environment production --json
```

Flags:

| Flag | Values | Default | Notes |
|------|--------|---------|-------|
| `--environment` | `draft`, `preview`, `production` | `draft` | Maps to Poof environments: draft/development, preview/mainnet-preview, production/live. |
| `--range` | `1h`, `6h`, `24h`, `3d`, `7d` | `24h` | Time window queried from Cloudflare Analytics Engine. |
| `--limit` | `1`-`50` | `10` | Max rows for top pages, failures, devices, countries, and referrers. |

Text output summarizes events, page views, route views, visitors, sessions, failure counts, and RUM
averages. JSON output includes:

```json
{
  "projectId": "...",
  "environment": "production",
  "siteIds": ["my-app", "worker-id"],
  "dataset": "poof_client_app_events",
  "timeRange": { "start": "...", "end": "...", "range": "24h" },
  "summary": {
    "events": 0,
    "pageViews": 0,
    "routeViews": 0,
    "visitors": 0,
    "sessions": 0,
    "errors": 0,
    "apiErrors": 0,
    "resourceErrors": 0,
    "jsErrors": 0,
    "averageTtfbMs": 0,
    "averageLcpMs": 0,
    "averageInpMs": 0,
    "averageCls": 0
  },
  "timeSeries": [],
  "topPages": [],
  "errors": [],
  "devices": [],
  "countries": [],
  "referrers": [],
  "metadata": { "dataSource": "cloudflare_analytics_engine", "fetchedAt": "..." }
}
```

`poof logs` and `poof analytics` answer different questions:

- Use `poof logs` for backend/API Worker request logs.
- Use `poof analytics` for client-side runtime failures, failed static assets, failed browser API calls,
  page/route traffic, RUM, and edge serving failures.

## MCP Retrieval

MCP clients can call `get_client_app_analytics`.

Project-scoped MCP:

```json
{
  "name": "get_client_app_analytics",
  "arguments": {
    "environment": "preview",
    "range": "1h",
    "limit": 10
  }
}
```

Top-level Poof MCP:

```json
{
  "name": "get_client_app_analytics",
  "arguments": {
    "projectId": "<project-id>",
    "environment": "production",
    "range": "24h",
    "limit": 10
  }
}
```

The MCP response has the same shape as `poof analytics --json`.

## Custom App Events

Generated apps and local frontends do not need their own provider for default telemetry. If an app
needs a small app-specific event later, call the injected helper instead of adding Google Analytics,
PostHog, or another provider:

```js
window.PoofAnalytics?.track('checkout_started', {
  plan: 'pro',
  source: 'pricing-page'
});
```

Custom event payloads are sanitized and written as `custom_event` rows. Keep props low-cardinality and
non-sensitive. Never pass wallet addresses, emails, auth tokens, request bodies, response bodies, or
user-generated content.

## Agent Debugging Pattern

When a deployed app is reported as broken:

1. Run `poof usage status -p <id>` first if the symptom is a 403 or "application disabled".
2. Run `poof analytics -p <id> --environment <draft|preview|production> --range 1h --json` to check
   browser errors, failed API/resource loads, 4xx/5xx events, and RUM regressions.
3. Run `poof logs -p <id> --environment <env>` for backend/API Worker logs if analytics shows API
   failures or edge dispatch errors.
4. Use `poof project status -p <id> --json` to confirm deploy URLs and environment state.
5. Fix via `poof iterate` or local source changes, redeploy, and re-check analytics for new failures.

Do not ask generated apps to add their own analytics provider for default telemetry. Poof's edge
analytics already covers generated and statically deployed client apps.
