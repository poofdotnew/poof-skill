# Backend-Only Mode

Use `generationMode: "backend,policy"` when you're building your own frontend locally but want Poof to generate and host the backend — API routes, database policies, and the PartyServer.

## Contents
- [When to Use](#when-to-use)
- [Creating a Backend-Only Project](#creating-a-backend-only-project)
- [Getting Connection Info](#getting-connection-info)
- [Setting Up Your Local Frontend](#setting-up-your-local-frontend)
- [Connecting to the PartyServer Backend](#connecting-to-the-partyserver-backend)
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

```typescript
const projectId = uuidv4();
const tarobaseToken = await getIdToken();

await mcpCall('tools/call', {
  name: 'create_project',
  arguments: {
    projectId,
    firstMessage: 'Build a task management backend with user profiles, project boards, and task assignments. Off-chain storage, owner-only writes for profiles, project members can create/update tasks.',
    tarobaseToken,
    isPublic: true,
    generationMode: 'backend,policy',
  },
});

await pollUntilDone(projectId);
```

The Poof AI will generate:
- **Database policies** — schema, access rules, hooks, queries
- **Backend API routes** — hosted on the PartyServer
- **Lifecycle actions** — for testing

It will **not** generate any frontend UI code.

## Getting Connection Info

After the build completes, call `get_project_status` to get the connection details your local frontend needs:

```typescript
const status = await mcpCall('tools/call', {
  name: 'get_project_status',
  arguments: { projectId },
});

// Connection info for your local frontend
const { connectionInfo } = status;

// Each environment has its own tarobaseAppId and backendUrl
// Draft is always available after create_project
console.log('Draft App ID:', connectionInfo.draft?.tarobaseAppId);
console.log('Draft Backend:', connectionInfo.draft?.backendUrl);

// Preview and production are available after deploying to those targets
console.log('Preview App ID:', connectionInfo.preview?.tarobaseAppId);
console.log('Production App ID:', connectionInfo.production?.tarobaseAppId);

// Shared URLs (same across all environments)
console.log('WebSocket URL:', connectionInfo.wsUrl);
console.log('API URL:', connectionInfo.apiUrl);
console.log('Auth API URL:', connectionInfo.authApiUrl);
```

The `connectionInfo` object contains per-environment connection details:

| Field | Value | Purpose |
|-------|-------|---------|
| `draft` | `{ tarobaseAppId, backendUrl }` or `null` | Draft/Poofnet environment (created immediately) |
| `preview` | `{ tarobaseAppId, backendUrl }` or `null` | Mainnet Preview environment (after `publish_project` with `target: "preview"`) |
| `production` | `{ tarobaseAppId, backendUrl }` or `null` | Production environment (after `publish_project` with `target: "production"`) |
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

// connectionInfo from get_project_status — pick your environment
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

Use `chat` to modify the backend just like any other project:

```typescript
await mcpCall('tools/call', {
  name: 'chat',
  arguments: {
    projectId,
    message: 'Add a notification system — when a task is assigned, create a notification record for the assignee. Add a GET /api/notifications endpoint that returns unread notifications.',
    messageId: uuidv4(),
    tarobaseToken: await getIdToken(),
  },
});

await pollUntilDone(projectId);
```

You can also use `steer_ai` to redirect the AI mid-execution, or `update_files` to directly modify backend files.

## Extracting the Generated SDK

If you have a Premium membership, you can download the generated typed SDK to use in your local project:

```typescript
// Start the download
const download = await mcpCall('tools/call', {
  name: 'download_code',
  arguments: { projectId, tarobaseToken: await getIdToken() },
});

// Poll until the download task completes
// ... poll get_task with download.taskId ...

// Get the signed URL
const url = await mcpCall('tools/call', {
  name: 'get_download_url',
  arguments: { projectId, taskId: download.taskId },
});

console.log('Download URL:', url.downloadUrl);
```

The zip contains:
- `db-client/` — typed database client
- `collections/` — collection definitions with TypeScript types
- Backend source files (API routes, policies)

Extract the `db-client` and `collections` directories into your local frontend project for type-safe database access.

## End-to-End Example

Complete workflow: create backend → get connection info → set up local frontend:

```typescript
import 'dotenv/config';
import { init, getIdToken } from '@pooflabs/server';
import { v4 as uuidv4 } from 'uuid';

// ... mcpCall and pollUntilDone helpers from building-and-chat.md ...

async function main() {
  // 1. Create a backend-only project
  const projectId = uuidv4();
  await mcpCall('tools/call', {
    name: 'create_project',
    arguments: {
      projectId,
      firstMessage: 'Build a social feed backend: user profiles (off-chain, owner-write), posts (off-chain, authenticated-create, public-read), likes (off-chain, authenticated-create/delete). Add API routes for feed pagination and trending posts.',
      tarobaseToken: await getIdToken(),
      isPublic: true,
      generationMode: 'backend,policy',
    },
  });
  await pollUntilDone(projectId);
  console.log('Backend build complete');

  // 2. Generate and run tests
  await mcpCall('tools/call', {
    name: 'chat',
    arguments: {
      projectId,
      message: 'Generate and run lifecycle action tests for all policies.',
      messageId: uuidv4(),
      tarobaseToken: await getIdToken(),
    },
  });
  await pollUntilDone(projectId);

  // 3. Get connection info for local frontend
  const status = await mcpCall('tools/call', {
    name: 'get_project_status',
    arguments: { projectId },
  });

  const { connectionInfo } = status;
  console.log('\n--- Connection Info for Local Frontend ---');
  console.log(`Tarobase App ID: ${connectionInfo.tarobaseAppId}`);
  console.log(`Backend URL:     ${connectionInfo.backendUrl}`);
  console.log(`WebSocket URL:   ${connectionInfo.wsUrl}`);
  console.log(`Auth API URL:    ${connectionInfo.authApiUrl}`);

  console.log('\n--- Local Frontend Setup ---');
  console.log('1. npm install @pooflabs/web');
  console.log('2. In your app entry point:');
  console.log(`   import { init, login, getDB } from '@pooflabs/web';`);
  console.log(`   init({ appId: '${connectionInfo.tarobaseAppId}' });`);
  console.log('3. Call login() before any authenticated operations');
  console.log(`4. Backend API routes available at: ${connectionInfo.backendUrl}/api/...`);
}

main().catch(console.error);
```
