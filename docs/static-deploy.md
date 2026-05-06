# Static Frontend Deploy

Deploy a self-built static frontend to a Poof project. Use this when you build the UI locally (React, Vue, Svelte, plain HTML, etc.) and want to host it on Poof alongside your backend.

## When to Use This

Use `poof deploy static` when:
- You're building your own frontend outside of Poof (e.g. a custom React/Vue/Svelte app)
- You used `--mode policy` or `--mode backend,policy` and want to deploy your local UI
- You have a pre-built `dist/` or `build/` folder ready to upload

Don't use this when:
- You're using Poof's AI to generate the UI (`--mode full` or `--mode ui,policy`) — Poof handles deployment automatically
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

# 2. Verify the backend (lifecycle tests only — auto-detected for backend-only)
poof verify -p $PROJECT_ID

# 3. Build frontend locally using connectionInfo from `poof project status`
cd ./my-frontend && npm run build && cd ..
tar czf dist.tar.gz -C ./my-frontend/dist .

# 4. If you want Poof-hosted UI lifecycle tests, upload source-authored test files
#    (keys are project paths like lifecycle-actions/ui-test-create-post.json)
#    See testing.md#how-to-generate-ui-test-json for how to author them.
poof files update -p $PROJECT_ID --from-json lifecycle-ui-tests.json

# 5. Deploy static frontend — your UI now lives at the draft URL
poof deploy static -p $PROJECT_ID --archive dist.tar.gz --title "Initial frontend deploy"

# 6. Run uploaded Poof UI lifecycle tests against the real static frontend
poof verify -p $PROJECT_ID --ui-tests=true -m "Run the existing source-authored lifecycle-actions/ui-test-*.json files against the deployed draft app. Do not create or rewrite UI tests from the dist bundle."

# 7. Optionally also run agent-local browser smoke tests against the draft URL.
#    See docs/backend-only.md#testing-a-static-deploy for Playwright, claude-in-chrome, and curl recipes.

# 8. Inspect first-party client analytics/failures from the Cloudflare edge.
poof analytics -p $PROJECT_ID --environment draft --range 1h --json
```

**Do not use `poof verify --ui-tests=true` to generate UI tests from a statically-deployed
frontend.** Poof's AI cannot write meaningful tests against a minified `dist/` bundle it didn't
build. Use `--ui-tests=true` only after the local agent has authored `lifecycle-actions/ui-test-*.json`
from the real frontend source, uploaded those files with `poof files update`, and deployed the static
frontend. The prompt should tell Poof to run the existing source-authored tests, not create or rewrite
them from the bundle.

**Ordering matters.** Run `poof verify` BEFORE the static deploy to gate the backend policies
(it auto-detects `backend,policy` mode and skips UI tests). Run Poof-hosted UI lifecycle tests or
agent-local browser smoke tests AFTER the static deploy so they hit your real frontend instead of
Poof's placeholder shell. Treat failing UI tests the same as a failing `poof verify`: block
`poof ship` until the source is fixed, rebuilt, redeployed, and rerun.

## How It Works

1. **Upload** — Your tar.gz is uploaded to S3 via presigned URL (`projects/{projectId}/static-dist/{taskId}.tar.gz`)
2. **Extract** — The deploy service safely extracts the archive (with tar-slip and symlink protection)
3. **Publish** — Individual files are synced to Cloudflare R2 via `aws s3 sync`
4. **Serve** — Your frontend is available at `https://{slug}-preview.poof.new`

The tar.gz is only a transport format — it's never served directly. R2 receives the extracted individual files (`index.html`, JS bundles, etc.), which Cloudflare serves via wildcard DNS on `*.poof.new`.

Static deploys create a checkpoint task with `focusArea: 'static_deploy'`. The previous checkpoint's policy, constants, and API spec are preserved — so your backend and the static frontend coexist.

If the project has an active built backend artifact from `poof deploy backend`, the static deploy also
preserves that `backendBundleUrl`. Preview and production deploys then promote both active artifacts:
the uploaded static frontend and the uploaded PartyServer backend. See
[backend-artifact-deploy.md](backend-artifact-deploy.md) for the backend side of the lifecycle.

Default client analytics are added by Poof's Cloudflare proxy. You do not need to rebuild or add an
analytics SDK to your static frontend to get page/route traffic, browser errors, failed resources,
failed API calls, RUM, and edge failure metrics. Retrieve them with `poof analytics`; see
[analytics.md](analytics.md).

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
