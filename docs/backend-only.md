# Backend-Only Mode

Use `--mode backend,policy` when you're building your own frontend locally but want Poof to generate and host the backend — API routes, database policies, and the PartyServer.

## Contents
- [When to Use](#when-to-use)
- [Creating a Backend-Only Project](#creating-a-backend-only-project)
- [Getting Connection Info](#getting-connection-info)
- [Building Your Local Frontend](#building-your-local-frontend)
- [Iterating on the Backend](#iterating-on-the-backend)
- [Extracting the Generated SDK](#extracting-the-generated-sdk)
- [End-to-End Example](#end-to-end-example)

## When to Use

Backend-only mode is the right choice when:

- You're building a **React Native**, **Expo**, or **desktop** app with its own UI
- You want a **custom frontend framework** (Vue, Svelte, plain HTML) but need a hosted backend
- You need **full control over the UI** while Poof handles data, auth, and API routes
- You're integrating Poof's backend into an **existing frontend project**

Poof generates and hosts the backend (PartyServer with API routes + database policies). You build the frontend locally and connect to it using the `@pooflabs/web` SDK.

## Creating a Backend-Only Project

Use the AI path when you want Poof to design or modify the backend from a prompt:

```bash
poof build \
  -m "Build a task management backend with user profiles, project boards, and task assignments. Off-chain storage, owner-only writes for profiles, project members can create/update tasks." \
  --mode backend,policy \
  --public
```

The Poof AI will generate:
- **Database policies** — schema, access rules, hooks, queries
- **Backend API routes** — hosted on the PartyServer
- **Lifecycle actions** — for testing

It will **not** generate any frontend UI code.

Use the no-AI path when you already have local policy/constants files and want deterministic direct deployment:

```bash
poof project create --no-ai --title "Task Backend" --mode backend,policy
poof policy validate -p <project-id> --policy policy/poof.json --constants policy/constants.json
poof policy deploy -p <project-id> --policy policy/poof.json --constants policy/constants.json
```

The no-AI project is still a normal Poof project: use `poof project status -p <id> --json` for connection info, `poof data` for runtime reads/writes, and `poof iterate -p <id> -m "..."` later if you want Poof AI help on the same project.

## Getting Connection Info

After the build completes, retrieve connection details for your local frontend:

```bash
poof project status -p <project-id> --json | jq '.connectionInfo'
```

The `connectionInfo` object contains per-environment connection details:

| Field | Value | Purpose |
|-------|-------|---------|
| `draft` | `{ tarobaseAppId, backendUrl }` or `null` | Draft/Poofnet environment (created immediately) |
| `preview` | `{ tarobaseAppId, backendUrl }` or `null` | Mainnet Preview environment (after publishing with `target: "preview"`) |
| `production` | `{ tarobaseAppId, backendUrl }` or `null` | Production environment (after publishing with `target: "production"`) |
| `wsUrl` | WebSocket URL | Real-time data connection |
| `apiUrl` | Poof data-plane REST API URL | Database API |
| `authApiUrl` | Auth service URL | Pass to `init()` in `@pooflabs/web` |

Each environment has a separate Poof app with its own data. Use `draft` for development/testing, `preview` for mainnet testing with allowlisted wallets, and `production` for live. The legacy field name `tarobaseAppId` in the JSON above is what the server still emits — treat it as your per-environment Poof app ID.

## Building Your Local Frontend

> **Do not use CDN/esm.sh shortcuts for `@pooflabs/web`.** Always scaffold a real bundler project (Vite, Next.js, Remix, etc.) and `npm install @pooflabs/web buffer`. Loading the SDK from a CDN in a single HTML file silently breaks the Buffer polyfill ordering and Solana wallet adapter interop. See [local-frontend-guide.md#do-not-use-cdn--esmsh--unpkg--skypack-shortcuts](local-frontend-guide.md#do-not-use-cdn--esmsh--unpkg--skypack-shortcuts) for why.

See **[local-frontend-guide.md](local-frontend-guide.md)** for the complete guide to building a frontend that connects to a Poof backend. It covers:

- SDK installation, Buffer polyfill, and `init()` configuration
- Wallet authentication (Phantom, `useAuth` hook, `getIdToken()`)
- Direct database access (`get`, `set`, `subscribe`)
- Calling PartyServer API routes with auth headers
- Using the generated typed SDK (`db-client` + `collections/`)
- Real-time subscriptions with the `useRealtimeData` React hook
- Gotchas: timestamps, token decimals, auth loading states
- Complete React and vanilla JS examples

Quick start:

```bash
npm install @pooflabs/web buffer
```

```typescript
import { Buffer } from 'buffer';
globalThis.Buffer = Buffer;
window.Buffer = Buffer;

import { init, login, get, set } from '@pooflabs/web';

// connectionInfo from `poof project status` — pick your environment
const env = connectionInfo.draft; // or .preview, .production
await init({
  appId: env.tarobaseAppId,
  authApiUrl: connectionInfo.authApiUrl,
  apiUrl: connectionInfo.apiUrl,
  wsApiUrl: connectionInfo.wsUrl,
  authMethod: 'phantom',
  skipBackendInit: true,
});

await login();  // opens Phantom wallet prompt

// Read/write data through the policy engine
const notes = await get('notes');
await set('notes/note-1', { title: 'Hello', authorAddress: user.address });

// Call PartyServer API routes
const res = await fetch(`${env.backendUrl}/api/notes`);
```

## Iterating on the Backend

Use `poof iterate` to modify the backend:

```bash
poof iterate -p <project-id> \
  -m "Add a notification system — when a task is assigned, create a notification record for the assignee. Add a GET /api/notifications endpoint that returns unread notifications."
```

## Extracting the Generated SDK

If you have purchased credits, you can download the generated typed SDK to use in your local project:

```bash
# Start the download
poof deploy download -p <project-id>

# Get the signed download URL (use the taskId from the previous command's output)
poof deploy download-url -p <project-id> --task <taskId>
```

The zip contains:
- `db-client/` — typed database client
- `collections/` — collection definitions with TypeScript types
- Backend source files (API routes, policies)

Extract the `db-client` and `collections` directories into your local frontend project for type-safe database access.

## Verification Ordering (Important)

`poof verify` auto-detects backend-only projects (`generationMode=policy` or `backend,policy`)
and sends a lifecycle-only verification prompt. It will NOT generate UI functional tests, for
two reasons:

1. **Before a static deploy**, the draft URL serves a Poof placeholder shell (`<title>Poof Preview App</title>`).
   Any UI test run against it passes vacuously ("heading visible, interactive elements present")
   without actually testing your feature.
2. **After a static deploy**, Poof's AI only has access to your minified `dist/` bundle on the
   server — not the TypeScript/JSX source it was built from. It cannot reliably write meaningful
   UI functional tests from minified JS. When it tries, it falls back to the same vacuous DOM
   shape checks.

**Correct order for backend-only projects:**

1. `poof build --mode backend,policy` — backend + policies
2. `poof project status -p <id> --json` — capture `connectionInfo` and the draft URL
3. `poof verify -p <id>` — lifecycle tests only (auto-detected from generationMode)
4. Build your local frontend wired to `connectionInfo`, `npm run build`
5. Author and upload source-aware `lifecycle-actions/ui-test-*.json` files if you want Poof-hosted
   UI lifecycle tests for the local frontend
6. `poof deploy static -p <id> --archive dist.tar.gz` — your UI is now live at the draft URL
7. **UI verification.** Run the uploaded Poof UI lifecycle tests, agent-local browser tests, or both — see
   [Testing a Static Deploy](#testing-a-static-deploy) below
8. `poof ship -p <id>` — preview / production deploy

Do not use `poof verify --ui-tests=true` as a request for Poof's AI to generate UI tests from the
static bundle. Use it only after the external agent has authored source-aware `ui-test-*.json` files,
uploaded them to the project, and deployed the real static frontend.

## Testing a Static Deploy

After `poof deploy static`, your frontend is live at the project's draft URL (from `poof project status`
under `.urls.draft`). Testing it is the agent's job because the agent already has the real source
code checked out locally, so it knows the feature contract, routes, and selectors. Poof's AI only
sees your minified `dist/` bundle, so it should run source-authored tests, not invent assertions
from the deployed assets.

There are two useful UI verification layers:

1. **Poof-hosted UI lifecycle tests** — source-aware `lifecycle-actions/ui-test-*.json` files that
   the external agent writes and uploads, then Poof runs in its Browserbase/Stagehand runner.
2. **Agent-local browser tests** — Playwright, Cypress, `claude-in-chrome`, or a raw curl fallback
   that the external agent runs directly against the draft URL.

### Poof-Hosted UI Lifecycle Tests

Use this path when you want UI test results recorded in `poof task test-results` after a static
deploy.

1. Write test files from the local UI source, using exact visible text and feature contracts. The
full authoring recipe is in [testing.md#how-to-generate-ui-test-json](testing.md#how-to-generate-ui-test-json):

```json
{
  "version": 1,
  "name": "test-ui-create-note",
  "type": "ui-test",
  "description": "Create a note through the statically deployed UI",
  "steps": [
    {
      "act": "Click the 'New Note' button",
      "verify": {
        "extract": "Is the note editor visible?",
        "schema": { "editorVisible": "boolean" },
        "expect": { "editorVisible": true }
      }
    },
    {
      "act": "Type 'Launch checklist' in the title field and click 'Save'",
      "verify": {
        "extract": "The note titles visible in the notes list",
        "schema": { "titles": "string[]" },
        "expect": { "titles": { "contains": "Launch checklist" } }
      }
    }
  ]
}
```

2. Upload them as project files. The JSON map keys are Poof project paths:

```json
{
  "lifecycle-actions/ui-test-create-note.json": "{ ... file contents ... }"
}
```

```bash
poof files update -p <project-id> --from-json lifecycle-ui-tests.json
```

3. Deploy the frontend:

```bash
npm run build
tar czf dist.tar.gz -C dist .
poof deploy static -p <project-id> --archive dist.tar.gz
```

4. Force UI execution with an explicit "run existing tests" prompt:

```bash
poof verify -p <project-id> --ui-tests=true -m "Run the existing source-authored lifecycle-actions/ui-test-*.json files against the deployed draft app. Do not create, rewrite, or replace UI tests from the dist bundle. If a UI test fails, report the failing file and step instead of weakening the assertion."
```

5. Gate on fresh results:

```bash
poof task test-results -p <project-id> --json
```

If no fresh UI results appear, inspect `poof project messages -p <id> --limit 100 --json` to confirm
whether `run_all_ui_tests` was invoked. Treat missing results as a blocker, not a pass.

### Agent-Local Browser Smoke Tests

**What to check, at minimum:**

| Check | Why |
|---|---|
| HTTP 200 on the draft URL | Sanity: the deploy is actually serving |
| Page title matches your HTML, not "Poof Preview App" | Confirms your build replaced the placeholder shell |
| Your top-level heading/root element renders | Confirms the JS bundle executed without runtime errors |
| Browser console has zero errors (no `ReferenceError`, no CORS/CSP/import failures) | Catches bundler misconfig (Buffer polyfill, CommonJS interop, missing peers) |
| SDK initialization log ("SDK initialized (appId=...)") | Confirms `@pooflabs/web` wired up to the right Poof app |
| A read-path smoke call (e.g. `get('notes')`) returns without throwing | Confirms the frontend can actually reach the backend with the right appId/endpoints |
| Public-read collections render without wallet auth | Confirms the policy read rules are public |
| Auth-gated write fails gracefully without a wallet session | Confirms the policy engine is actually enforcing, and your UI degrades cleanly |

A wallet-signed E2E (Phantom connect, signed write, round-trip read) is usually out of scope for
automation because mainnet wallets require human signing. Exercise the signing paths manually or
gate them behind an "agent test mode" in your app.

**Recipes by tooling:**

### Claude Code with `claude-in-chrome` tools

```
1. tabs_context_mcp + tabs_create_mcp → get a fresh tab
2. navigate tab → https://<slug>-preview.poof.new
3. read_console_messages (pattern: ".", limit: 100) → assert no EXCEPTION entries;
   assert a "SDK initialized" log line from your app code
4. read_page (filter: "interactive") → assert your top-level heading + key buttons exist
5. For a read smoke test, use javascript_tool to run a small `get('notes')` via the SDK and
   log the result, then read_console_messages for that log line
6. Exit non-zero and block the deploy flow if any of the above fail
```

The `claude-in-chrome` tools are available inside the Claude skill runtime; no extra install needed.

### Playwright (any agent or CI)

```bash
npx playwright install chromium  # once
npx playwright test tests/draft-smoke.spec.ts
```

```ts
// tests/draft-smoke.spec.ts
import { test, expect } from '@playwright/test';

const DRAFT_URL = process.env.DRAFT_URL!; // from `poof project status -p <id> --json`

test('draft serves the real static frontend', async ({ page }) => {
  const errors: string[] = [];
  page.on('pageerror', (err) => errors.push(err.message));
  page.on('console', (msg) => { if (msg.type() === 'error') errors.push(msg.text()); });

  await page.goto(DRAFT_URL);
  await expect(page).toHaveTitle(/Wallet Auth Notes/); // NOT "Poof Preview App"
  await expect(page.getByRole('heading', { level: 1 })).toBeVisible();

  // SDK init log must appear within a few seconds of page load
  await page.waitForFunction(() =>
    performance.getEntriesByType('resource').some((r) => r.name.includes('/assets/')),
    { timeout: 5000 },
  );

  expect(errors, `console errors:\n${errors.join('\n')}`).toEqual([]);
});
```

A failing browser smoke test must be treated the same as a failing `poof verify`: block the `poof ship`
call, surface the error, fix the source, rebuild, redeploy, rerun. Do not report the flow as complete
with pending browser-side failures.

### Raw curl (minimum viable fallback)

```bash
# Smoke: HTTP 200 + title substring check
curl -sSf -o /tmp/draft.html https://<slug>-preview.poof.new
grep -q '<title>Wallet Auth Notes' /tmp/draft.html || { echo "title mismatch (placeholder?)"; exit 1; }
test $(wc -c </tmp/draft.html) -gt 1000 || { echo "body too small (placeholder?)"; exit 1; }
```

This catches the "deploy silently reverted to placeholder" case but nothing else. Prefer a real
browser runner for anything past basic sanity.

> Do not ask Poof's AI to write these tests from the deployed bundle via a generic
> `poof iterate -m "generate UI tests"` prompt. Either upload source-authored `ui-test-*.json`
> files and ask Poof to run those existing files, or run browser tests locally.

## End-to-End Example

Complete workflow: create backend, verify policies, build local frontend, deploy static, run source-aware UI tests, ship.

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Create a backend-only project with AI
PROJECT_ID=$(poof build \
  -m "Build a social feed backend: user profiles (off-chain, owner-write), posts (off-chain, authenticated-create, public-read), likes (off-chain, authenticated-create/delete). Add API routes for feed pagination and trending posts." \
  --mode backend,policy \
  --public \
  --json | jq -r '.projectId')
echo "Project: $PROJECT_ID"

# No-AI alternative:
# PROJECT_ID=$(poof project create --no-ai --title "Social Feed Backend" --mode backend,policy --json | jq -r '.projectId')
# poof policy deploy -p "$PROJECT_ID" --policy policy/poof.json --constants policy/constants.json

# 2. Capture connection info for your local frontend
poof project status -p "$PROJECT_ID" --json | jq '.connectionInfo' > connection-info.json
cat connection-info.json

# 3. Verify the backend (auto-detected as backend-only → lifecycle tests only)
poof verify -p "$PROJECT_ID"

# 4. Build your local frontend — wire @pooflabs/web using connectionInfo
#    (see local-frontend-guide.md for init() setup)
cd ./my-frontend && npm run build && cd ..
tar czf dist.tar.gz -C ./my-frontend/dist .

# 5. Upload source-authored Poof UI lifecycle tests
poof files update -p "$PROJECT_ID" --from-json lifecycle-ui-tests.json

# 6. Deploy the static frontend to the draft URL
poof deploy static -p "$PROJECT_ID" --archive dist.tar.gz --title "initial static deploy"

# 7. Run the uploaded tests against the real static UI
poof verify -p "$PROJECT_ID" --ui-tests=true -m "Run the existing source-authored lifecycle-actions/ui-test-*.json files against the deployed draft app. Do not create or rewrite UI tests from the dist bundle."

# 8. Ship to preview / production
poof ship -p "$PROJECT_ID"
```

See [Setting Up Your Local Frontend](#building-your-local-frontend) for the full SDK setup.
