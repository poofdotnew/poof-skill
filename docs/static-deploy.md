# Static Frontend Deploy

Deploy a self-built static frontend to a Poof project. Use this when you build the UI locally (React, Vue, Svelte, plain HTML, etc.) and want to host it on Poof alongside your backend.

## Contents
- [When to Use This](#when-to-use-this)
- [Prerequisites](#prerequisites)
- [API Reference](#api-reference)
- [MCP Tool](#mcp-tool)
- [CLI Command](#cli-command)
- [Step-by-Step Workflow](#step-by-step-workflow)
- [Full Example](#full-example)
- [How It Works](#how-it-works)
- [Constraints](#constraints)
- [Error Handling](#error-handling)

## When to Use This

Use `deploy-static` when:
- You're building your own frontend outside of Poof (e.g. a custom React/Vue/Svelte app)
- You used `generationMode: 'policy'` or `'backend,policy'` and want to deploy your local UI
- You have a pre-built `dist/` or `build/` folder ready to upload

Don't use this when:
- You're using Poof's AI to generate the UI (`generationMode: 'full'` or `'ui,policy'`) — Poof handles deployment automatically
- You only need the backend — use `publish_project` instead

## Prerequisites

1. An existing Poof project (create one with `create_project`)
2. A built frontend in a `dist/` folder (e.g. `npm run build`)
3. Standard Poof auth (Cognito JWT + wallet address)

## API Reference

### REST Endpoint

```
POST /api/project/{projectId}/deploy-static
```

### Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | `Bearer <cognito-id-token>` |
| `X-Wallet-Address` | Yes | Your Solana wallet address |
| `Content-Type` | Yes | `application/gzip` |
| `X-Deploy-Title` | No | Checkpoint title (default: "Static frontend deploy") |
| `X-Deploy-Description` | No | Checkpoint description |

### Request Body

Raw `tar.gz` bytes. Create the archive from your dist folder:

```bash
tar czf dist.tar.gz -C dist .
```

### Success Response (200)

```json
{
  "success": true,
  "data": {
    "projectId": "uuid",
    "taskId": "uuid",
    "bundleUrl": "https://your-slug-preview.poof.new",
    "slug": "your-project-slug"
  }
}
```

## MCP Tool

The `deploy_static_frontend` MCP tool provides the same functionality via JSON-RPC. Since the MCP protocol uses JSON, the archive must be **base64-encoded**.

```typescript
await mcpCall('tools/call', {
  name: 'deploy_static_frontend',
  arguments: {
    projectId: 'your-project-id',
    archiveBase64: fs.readFileSync('dist.tar.gz').toString('base64'),
    title: 'v1.2.0 release',           // optional
    description: 'Updated dashboard',   // optional
  },
});
```

### Response Shape

```json
{
  "projectId": "uuid",
  "taskId": "uuid",
  "bundleUrl": "https://your-slug-preview.poof.new",
  "slug": "your-project-slug"
}
```

## CLI Command

```bash
poof deploy static -p <project-id> --archive dist.tar.gz
```

### Flags

| Flag | Required | Description |
|------|----------|-------------|
| `--archive` | Yes | Path to `.tar.gz` file |
| `--title` | No | Checkpoint title |
| `--description` | No | Checkpoint description |
| `--dry-run` | No | Validate without deploying |

### Examples

```bash
# Build and deploy in one shot
npm run build && tar czf dist.tar.gz -C dist . && poof deploy static -p $PROJECT_ID --archive dist.tar.gz

# With custom title
poof deploy static -p $PROJECT_ID --archive dist.tar.gz --title "v2.0 release"

# Dry run to validate
poof deploy static -p $PROJECT_ID --archive dist.tar.gz --dry-run
```

## Step-by-Step Workflow

### 1. Create a project with backend-only mode

```typescript
await mcpCall('tools/call', {
  name: 'create_project',
  arguments: {
    projectId: uuidv4(),
    firstMessage: 'Build a token staking backend with...',
    tarobaseToken: idToken,
    generationMode: 'backend,policy',
    isPublic: true,
  },
});
await pollUntilDone(projectId);
```

### 2. Build your frontend locally

```bash
cd my-frontend
npm run build   # produces dist/ folder
```

### 3. Create the tar.gz archive

```bash
tar czf dist.tar.gz -C dist .
```

The archive must contain your built files at the root level (not inside a subdirectory).

### 4. Upload via REST API

```typescript
const archive = fs.readFileSync('dist.tar.gz');

const response = await fetch(
  `${baseUrl}/api/project/${projectId}/deploy-static`,
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${idToken}`,
      'X-Wallet-Address': walletAddress,
      'Content-Type': 'application/gzip',
      'X-Deploy-Title': 'Initial deploy',
    },
    body: archive,
  }
);

const result = await response.json();
console.log('Live at:', result.data.bundleUrl);
```

### 5. Verify deployment

```typescript
const status = await mcpCall('tools/call', {
  name: 'get_project_status',
  arguments: { projectId },
});
console.log('URLs:', status.urls);
```

## Full Example

End-to-end TypeScript script — creates a project, builds a frontend, and deploys it:

```typescript
import 'dotenv/config';
import { init, getIdToken } from '@pooflabs/server';
import { v4 as uuidv4 } from 'uuid';
import { execSync } from 'child_process';
import fs from 'fs';

// Setup auth
init({ appId: '697d5189a1e3dd2cc1a82d2b', authApiUrl: 'https://auth.tarobase.com' });
process.env.TAROBASE_SOLANA_KEYPAIR = process.env.SOLANA_PRIVATE_KEY!;

const walletAddress = process.env.SOLANA_WALLET_ADDRESS!;
const baseUrl = 'https://poof.new';
const projectId = uuidv4();

async function main() {
  const idToken = await getIdToken();

  // 1. Create project with backend-only mode
  await mcpCall('tools/call', {
    name: 'create_project',
    arguments: {
      projectId,
      firstMessage: 'Build a token staking backend with deposit, withdraw, and claim-rewards actions.',
      tarobaseToken: idToken,
      generationMode: 'backend,policy',
      isPublic: true,
    },
  });
  await pollUntilDone(projectId);

  // 2. Build the frontend
  execSync('npm run build', { cwd: './my-frontend', stdio: 'inherit' });

  // 3. Create archive
  execSync('tar czf dist.tar.gz -C ./my-frontend/dist .', { stdio: 'inherit' });

  // 4. Deploy static frontend
  const archive = fs.readFileSync('dist.tar.gz');
  const deployRes = await fetch(`${baseUrl}/api/project/${projectId}/deploy-static`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${await getIdToken()}`,
      'X-Wallet-Address': walletAddress,
      'Content-Type': 'application/gzip',
      'X-Deploy-Title': 'Initial frontend deploy',
    },
    body: archive,
  });

  if (!deployRes.ok) {
    throw new Error(`Deploy failed: ${await deployRes.text()}`);
  }

  const result = await deployRes.json();
  console.log('Deployed:', result.data.bundleUrl);

  // Cleanup
  fs.unlinkSync('dist.tar.gz');
}

main().catch(console.error);
```

## How It Works

1. **Upload** — Your tar.gz is uploaded to S3 (`projects/{projectId}/static-dist/{taskId}.tar.gz`)
2. **Extract** — The dev-server-manager safely extracts the archive (with tar-slip and symlink protection)
3. **Publish** — Files are synced to Cloudflare R2 via `publishBuild()`
4. **Serve** — Your frontend is available at `https://{slug}-preview.poof.new`

Static deploys create a checkpoint task with `focusArea: 'static_deploy'`. The previous checkpoint's `codeDeltaUrl`, policy, constants, and API spec are preserved — so your backend code and the static frontend coexist.

## Constraints

| Constraint | Value |
|-----------|-------|
| Max archive size | 50 MB (gzip-compressed) |
| Max extracted size | 500 MB |
| Archive format | `.tar.gz` only (gzip magic bytes `0x1f 0x8b` validated) |
| Deployment tier | Preview only (`{slug}-preview.poof.new`) |
| Archive contents | Files at root level (no wrapping subdirectory) |

## Error Handling

| Status | Cause | Fix |
|--------|-------|-----|
| 400 | Empty body or invalid archive format | Send a valid `tar.gz` created with `tar czf dist.tar.gz -C dist .` |
| 401 | Invalid or expired JWT | Refresh with `getIdToken()` |
| 403 | Not the project owner | Check project ownership — only the creator can deploy |
| 404 | Project not found | Verify the project ID exists |
| 413 | Archive exceeds 50 MB | Reduce bundle size (tree-shake, compress assets, remove source maps) |
| 500 | S3 upload or R2 sync failed | Retry the request |
| 503 | Deploy service not configured | Internal error — contact Poof team |
