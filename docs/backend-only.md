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
| `apiUrl` | Tarobase REST API URL | Database API |
| `authApiUrl` | Auth service URL | Pass to `init()` in `@pooflabs/web` |

Each environment has a separate Tarobase app with its own data. Use `draft` for development/testing, `preview` for mainnet testing with allowlisted wallets, and `production` for live.

## Building Your Local Frontend

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

## End-to-End Example

Complete workflow: create backend, get connection info, set up local frontend.

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Create a backend-only project
PROJECT_ID=$(poof build \
  -m "Build a social feed backend: user profiles (off-chain, owner-write), posts (off-chain, authenticated-create, public-read), likes (off-chain, authenticated-create/delete). Add API routes for feed pagination and trending posts." \
  --mode backend,policy \
  --public \
  --json | jq -r '.projectId')
echo "Project: $PROJECT_ID"

# 3. Iterate — generate and run tests
poof iterate -p "$PROJECT_ID" \
  -m "Generate and run lifecycle action tests for all policies."

# 4. Get connection info for your local frontend
poof project status -p "$PROJECT_ID" --json | jq '.connectionInfo'

echo ""
echo "--- Local Frontend Setup ---"
echo "1. npm install @pooflabs/web buffer"
echo "2. In your app entry point:"
echo "   import { init, login, get, set } from '@pooflabs/web';"
echo "   init({ appId: '<tarobaseAppId from above>', ... });"
echo "3. Call login() before any authenticated operations"
echo "4. Backend API routes available at: <backendUrl from above>/api/..."
```

After the script runs, wire up your local frontend with the `@pooflabs/web` SDK using the connection info printed in step 4. See [Setting Up Your Local Frontend](#building-your-local-frontend) for the full setup.
