# Static Frontend Deploy

Deploy a self-built static frontend to a Poof project. Use this when you build the UI locally (React, Vue, Svelte, plain HTML, etc.) and want to host it on Poof alongside your backend.

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

## REST API

Static deploy uses a presigned URL flow (3 steps):

### Step 1: Get upload URL

```
POST /api/project/{projectId}/deploy-static/upload-url
Content-Type: application/json
Authorization: Bearer <cognito-id-token>
X-Wallet-Address: <solana-wallet-address>

Body: { "title": "optional", "description": "optional" }
```

Returns:
```json
{
  "success": true,
  "data": {
    "uploadUrl": "https://s3.amazonaws.com/...presigned...",
    "taskId": "uuid",
    "s3Key": "projects/{projectId}/static-dist/{taskId}.tar.gz",
    "maxSize": 52428800,
    "expiresIn": 600
  }
}
```

### Step 2: Upload to S3

```
PUT {uploadUrl}
Content-Type: application/gzip

Body: raw tar.gz bytes
```

### Step 3: Trigger deploy

```
POST /api/project/{projectId}/deploy-static/trigger
Content-Type: application/json
Authorization: Bearer <cognito-id-token>
X-Wallet-Address: <solana-wallet-address>

Body: { "taskId": "uuid-from-step-1" }
```

Returns:
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

The `deploy_static_frontend` MCP tool handles all three steps internally. Since MCP uses JSON, the archive must be **base64-encoded**.

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

Response:
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

| Flag | Required | Description |
|------|----------|-------------|
| `--archive` | Yes | Path to `.tar.gz` file |
| `--title` | No | Checkpoint title |
| `--description` | No | Checkpoint description |
| `--dry-run` | No | Validate without deploying |

```bash
# Build and deploy
npm run build && tar czf dist.tar.gz -C dist . && poof deploy static -p $PROJECT_ID --archive dist.tar.gz

# With custom title
poof deploy static -p $PROJECT_ID --archive dist.tar.gz --title "v2.0 release"
```

## Workflow Example

```typescript
// 1. Create project with backend-only mode
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

// 2. Build frontend locally
execSync('npm run build', { cwd: './my-frontend', stdio: 'inherit' });
execSync('tar czf dist.tar.gz -C ./my-frontend/dist .', { stdio: 'inherit' });

// 3. Deploy via MCP (handles presigned URL flow internally)
const result = await mcpCall('tools/call', {
  name: 'deploy_static_frontend',
  arguments: {
    projectId,
    archiveBase64: fs.readFileSync('dist.tar.gz').toString('base64'),
    title: 'Initial frontend deploy',
  },
});
console.log('Live at:', result.bundleUrl);
```

## How It Works

1. **Upload** — Your tar.gz is uploaded to S3 via presigned URL (`projects/{projectId}/static-dist/{taskId}.tar.gz`)
2. **Extract** — The deploy service safely extracts the archive (with tar-slip and symlink protection)
3. **Publish** — Individual files are synced to Cloudflare R2 via `aws s3 sync`
4. **Serve** — Your frontend is available at `https://{slug}-preview.poof.new`

The tar.gz is only a transport format — it's never served directly. R2 receives the extracted individual files (`index.html`, JS bundles, etc.), which Cloudflare serves via wildcard DNS on `*.poof.new`.

Static deploys create a checkpoint task with `focusArea: 'static_deploy'`. The previous checkpoint's policy, constants, and API spec are preserved — so your backend and the static frontend coexist.

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
| 400 | Empty body, invalid archive, or missing taskId | Ensure valid `tar.gz` created with `tar czf dist.tar.gz -C dist .` |
| 401 | Invalid or expired JWT | Refresh with `getIdToken()` |
| 403 | Not the project owner | Only the project creator can deploy |
| 404 | Project or task not found | Verify project ID and task ID |
| 409 | Task already triggered | Each upload URL can only be triggered once |
| 413 | Archive exceeds 50 MB | Reduce bundle size (tree-shake, compress assets, remove source maps) |
| 500 | S3 upload or R2 sync failed | Retry the request |
