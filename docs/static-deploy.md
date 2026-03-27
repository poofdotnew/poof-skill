# Static Frontend Deploy

Deploy a self-built static frontend to a Poof project. Use this when you build the UI locally (React, Vue, Svelte, plain HTML, etc.) and want to host it on Poof alongside your backend.

## When to Use This

Use `deploy-static` when:
- You're building your own frontend outside of Poof (e.g. a custom React/Vue/Svelte app)
- You used `generationMode: 'policy'` or `'backend,policy'` and want to deploy your local UI
- You have a pre-built `dist/` or `build/` folder ready to upload

Don't use this when:
- You're using Poof's AI to generate the UI (`generationMode: 'full'` or `'ui,policy'`) — Poof handles deployment automatically
- You only need the backend — use `poof ship` instead

## Prerequisites

1. An existing Poof project (create one with `poof build`)
2. A built frontend in a `dist/` folder (e.g. `npm run build`)
3. Authenticated CLI (`poof auth login`)

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

```bash
# 1. Create project with backend-only mode
PROJECT_ID=$(poof build -m "Build a token staking backend with..." --mode backend,policy --quiet)

# 2. Build frontend locally
cd ./my-frontend && npm run build && cd ..
tar czf dist.tar.gz -C ./my-frontend/dist .

# 3. Deploy static frontend
poof deploy static -p $PROJECT_ID --archive dist.tar.gz --title "Initial frontend deploy"
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

| Error | Cause | Fix |
|-------|-------|-----|
| Empty/invalid archive | Empty body or bad format | Ensure valid `tar.gz` created with `tar czf dist.tar.gz -C dist .` |
| Not authenticated | Invalid or expired token | Run `poof auth login` |
| Not project owner | Wrong account | Only the project creator can deploy |
| Archive too large | Exceeds 50 MB | Reduce bundle size (tree-shake, compress assets, remove source maps) |

