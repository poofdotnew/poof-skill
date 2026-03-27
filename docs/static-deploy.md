# Static Frontend Deploy

Deploy a pre-built static frontend to Poof. This is for projects where you build the frontend yourself (e.g. with Vite, Next.js static export, plain HTML) and want to host it on Poof alongside your Poof-managed backend and on-chain infrastructure.

## Contents
- [When to Use This](#when-to-use-this)
- [Prerequisites](#prerequisites)
- [API Reference](#api-reference)
- [Step-by-Step Workflow](#step-by-step-workflow)
- [Full Example](#full-example)
- [How It Works](#how-it-works)
- [Constraints](#constraints)
- [Error Handling](#error-handling)

## When to Use This

Use the static deploy API when:

- You have a **locally-built frontend** (React, Vue, Svelte, plain HTML, etc.) that you want hosted on Poof
- You want your frontend **bundled alongside a Poof backend** — same project, same preview URL, same deployment pipeline
- You're building the UI yourself but using Poof for the backend, database policies, and on-chain features
- You generated a project with `generationMode: "backend,policy"` or `"policy"` and built your own frontend separately

Do **not** use this if Poof is already generating your frontend via `chat` with `generationMode: "full"` or `"ui,policy"` — in that case, use `publish_project` instead.

## Prerequisites

1. **An existing Poof project** — create one first via `create_project`. The static deploy endpoint does not support `projectId: "new"`.
2. **A built dist folder** — your frontend must be compiled/bundled into a directory of static assets (HTML, JS, CSS, images, fonts, etc.).
3. **Standard Poof auth** — Cognito JWT + wallet address. See [SKILL.md](../SKILL.md#authentication).

## API Reference

### `POST /api/project/{projectId}/deploy-static`

This is a **REST endpoint**, not an MCP tool. Call it directly with `fetch`.

**Headers:**

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | `Bearer <cognito-id-token>` |
| `X-Wallet-Address` | Yes | Solana wallet address |
| `Content-Type` | Yes | `application/gzip` |
| `X-Deploy-Title` | No | Deployment title (defaults to "Static frontend deploy") |
| `X-Deploy-Description` | No | Description of what changed |

**Body:** Raw `tar.gz` bytes of your dist folder contents.

**Success Response (200):**

```json
{
  "success": true,
  "data": {
    "projectId": "abc123",
    "taskId": "def456",
    "bundleUrl": "https://my-project-preview.poof.new",
    "slug": "my-project"
  }
}
```

## Step-by-Step Workflow

### 1. Create a Poof project (if you don't have one)

Create a backend-only project that your static frontend will connect to:

```typescript
const projectId = uuidv4();

await mcpCall('tools/call', {
  name: 'create_project',
  arguments: {
    projectId,
    firstMessage: 'Build a token-gated API with user profiles and content storage...',
    tarobaseToken: await getIdToken(),
    isPublic: true,
    generationMode: 'backend,policy',
  },
});

await pollUntilDone(projectId);
```

### 2. Build your frontend

Build your static frontend however you normally would:

```bash
npm run build
# Output goes to ./dist (or ./build, ./out, etc.)
```

### 3. Create a tar.gz archive

Package the dist folder contents into a gzip-compressed tar archive:

```typescript
import { execSync } from 'child_process';

// The -C flag means "change to this directory first" so the archive
// contains the files directly, not nested under a dist/ prefix
execSync('tar czf /tmp/dist.tar.gz -C dist .');
```

### 4. Upload to Poof

```typescript
import { readFileSync } from 'fs';
import { getIdToken } from '@pooflabs/server';

const archive = readFileSync('/tmp/dist.tar.gz');
const idToken = await getIdToken();

const response = await fetch(
  `${baseUrl}/api/project/${projectId}/deploy-static`,
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/gzip',
      'Authorization': `Bearer ${idToken}`,
      'X-Wallet-Address': walletAddress,
      'X-Deploy-Title': 'Frontend v1.0',
      'X-Deploy-Description': 'Initial static deploy',
    },
    body: archive,
  }
);

const result = await response.json();
console.log(`Deployed to: ${result.data.bundleUrl}`);
```

### 5. Poll the task (optional)

The response includes a `taskId`. Poll it to confirm the deploy completed:

```typescript
let task;
do {
  await new Promise(r => setTimeout(r, 3000));
  task = await mcpCall('tools/call', {
    name: 'get_task',
    arguments: { projectId, taskId: result.data.taskId },
  });
} while (task.status !== 'completed' && task.status !== 'failed');
```

## Full Example

End-to-end: create a backend project, build a local frontend, and deploy it.

```typescript
import 'dotenv/config';
import { execSync } from 'child_process';
import { readFileSync } from 'fs';
import { init, getIdToken } from '@pooflabs/server';
import { v4 as uuidv4 } from 'uuid';

// --- Auth setup (see building-and-chat.md for full mcpCall helper) ---
const POOF_ENVIRONMENTS = {
  production: {
    appId: '697d5189a1e3dd2cc1a82d2b',
    baseUrl: 'https://poof.new',
    authUrl: 'https://auth.tarobase.com',
  },
  staging: {
    appId: '6993d4b0b2b6ac08cd334dfb',
    baseUrl: 'https://staging.poof.new',
    authUrl: 'https://auth-staging.tarobase.com',
    vercelBypassToken: process.env.VERCEL_BYPASS_TOKEN,
  },
} as const;

type PoofEnv = keyof typeof POOF_ENVIRONMENTS;
const env = POOF_ENVIRONMENTS[(process.env.POOF_ENV as PoofEnv) || 'production'];

init({ appId: env.appId, authApiUrl: env.authUrl });
process.env.TAROBASE_SOLANA_KEYPAIR = process.env.SOLANA_PRIVATE_KEY!;
const walletAddress = process.env.SOLANA_WALLET_ADDRESS!;

// --- 1. Create a backend-only Poof project ---
const projectId = uuidv4();

await mcpCall('tools/call', {
  name: 'create_project',
  arguments: {
    projectId,
    firstMessage: 'Build a REST API for a token-gated content platform with user profiles',
    tarobaseToken: await getIdToken(),
    isPublic: true,
    generationMode: 'backend,policy',
  },
});

await pollUntilDone(projectId);

// --- 2. Build your static frontend ---
execSync('npm run build', { stdio: 'inherit' });

// --- 3. Package the dist folder ---
execSync('tar czf /tmp/dist.tar.gz -C dist .');

// --- 4. Deploy to Poof ---
const archive = readFileSync('/tmp/dist.tar.gz');
const idToken = await getIdToken();

const headers: Record<string, string> = {
  'Content-Type': 'application/gzip',
  'Authorization': `Bearer ${idToken}`,
  'X-Wallet-Address': walletAddress,
  'X-Deploy-Title': 'Initial frontend deploy',
  'X-Deploy-Description': 'Static React app bundled with Poof backend',
};

if ('vercelBypassToken' in env && env.vercelBypassToken) {
  headers['x-vercel-protection-bypass'] = env.vercelBypassToken;
}

const response = await fetch(
  `${env.baseUrl}/api/project/${projectId}/deploy-static`,
  {
    method: 'POST',
    headers,
    body: archive,
  }
);

if (!response.ok) {
  const error = await response.json();
  throw new Error(`Deploy failed (${response.status}): ${JSON.stringify(error)}`);
}

const { data } = await response.json();
console.log(`Deployed to: ${data.bundleUrl}`);

// --- 5. Poll task to confirm completion ---
let task;
do {
  await new Promise(r => setTimeout(r, 3000));
  task = await mcpCall('tools/call', {
    name: 'get_task',
    arguments: { projectId, taskId: data.taskId },
  });
} while (task.status !== 'completed' && task.status !== 'failed');

if (task.status === 'completed') {
  console.log('Deploy complete!');
} else {
  console.error('Deploy failed:', task);
}
```

## How It Works

1. The endpoint validates auth, then uploads the `tar.gz` to S3 at `projects/{projectId}/static-dist/{taskId}.tar.gz`
2. Poof's dev-server-manager downloads the archive, extracts it, and publishes the static assets to Cloudflare R2
3. The task is marked completed with a `bundleUrl` pointing to the preview URL
4. Your existing backend source code (`codeDeltaUrl`) is preserved — static deploys don't overwrite it

**Key design details:**

- **`staticDistUrl` vs `codeDeltaUrl`** — These coexist on a project. `codeDeltaUrl` points to source code archives (backend); `staticDistUrl` points to pre-built dist archives (frontend). A project can have both.
- **Always deploys to preview tier** — Production promotion happens through the normal Poof deployment flow.
- **Binary-safe** — The tar.gz format preserves images, fonts, WASM, and any other binary assets natively.

## Constraints

| Limit | Value |
|-------|-------|
| Max payload size | 50 MB |
| Archive format | `tar.gz` (gzip magic bytes `1f 8b` validated) |
| Project | Must exist first (use `create_project`) |
| Deploy tier | Preview only (promote to production via normal flow) |

## Error Handling

| Status | When | Fix |
|--------|------|-----|
| 400 | Empty body or invalid gzip format | Ensure body is raw `tar.gz` bytes with correct gzip header |
| 401 | Not authenticated | Refresh JWT via `getIdToken()` |
| 403 | Not the project owner | Check you're using the correct wallet/keypair |
| 404 | Project not found | Create the project first via `create_project` |
| 413 | Payload exceeds 50 MB | Reduce bundle size — check for unnecessary assets |
| 500 | S3 upload or R2 sync failed | Retry the request; if persistent, check Poof status |
