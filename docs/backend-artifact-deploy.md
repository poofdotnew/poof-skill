# Built Backend Artifact Deploy

Deploy a self-built PartyServer backend Worker to a Poof project. Use this when you build or own the backend bundle locally and want Poof to host it with Poof-owned Cloudflare credentials, dispatch namespace, platform bindings, secrets, queues, and heartbeat registration.

This is **built bundle mode**, not source upload. Upload Wrangler's bundled Worker output, not a raw `tsc` `dist/` folder. Raw TypeScript output often is not self-contained.

## When to Use This

Use `poof deploy backend` when:

- You built a PartyServer or Worker backend locally and want to deploy that exact bundle.
- You want to preserve a custom backend across static UI, preview, and production deploys.
- You are pairing a local frontend deploy (`poof deploy static`) with a local backend deploy.
- You want Poof to own the final Cloudflare deploy credentials and environment wiring.

Do not use this when:

- You want Poof's AI to edit backend source. Use `poof iterate` on a source-backed project instead.
- You only have raw TypeScript emitted by `tsc`. Bundle it with Wrangler first.
- You need to inspect or modify backend source from Poof after upload. Built bundles intentionally limit source visibility.

## Build and Package

From your local backend project, build the Worker bundle with Wrangler:

```bash
bunx wrangler deploy --dry-run --outdir .poof-backend-bundle
```

Add the Poof artifact manifest to the output directory:

```bash
cat > .poof-backend-bundle/poof-backend-artifact.json <<'JSON'
{
  "entrypoint": "index.js",
  "wranglerVersion": "4.80.0",
  "apiSpecPath": "generated/api-spec.json"
}
JSON
```

Archive the bundled output:

```bash
tar czf backend-worker.tar.gz -C .poof-backend-bundle .
```

Deploy it to the draft backend:

```bash
poof deploy backend -p <project-id> --archive backend-worker.tar.gz
```

Useful variants:

```bash
poof deploy backend -p <project-id> --archive backend-worker.tar.gz --title "backend v2"
poof deploy backend -p <project-id> --archive backend-worker.tar.gz --description "Adds queue consumers"
poof deploy backend -p <project-id> --archive backend-worker.tar.gz --dry-run
```

## Manifest Contract

The archive must include `poof-backend-artifact.json` at the archive root.

| Field | Required | Purpose |
|---|---:|---|
| `entrypoint` | Yes | Worker entrypoint inside the archive. `main` is accepted as a fallback for compatibility. |
| `wranglerVersion` | Yes | Wrangler version used to produce the bundled output. |
| `apiSpecPath` | No | Path to generated API metadata, usually `generated/api-spec.json`. |
| `queuesPath` | No | Path to queue config JSON if the backend uses Poof Cloud Queues. |
| `heartbeatPath` | No | Path to heartbeat/scheduled task config JSON. |

Example with optional platform metadata:

```json
{
  "entrypoint": "index.js",
  "wranglerVersion": "4.80.0",
  "apiSpecPath": "generated/api-spec.json",
  "queuesPath": "config/queues.json",
  "heartbeatPath": "config/heartbeat.json"
}
```

## Validation Rules

The CLI and server both validate the archive before deploying:

- Archive must be gzip-compressed tar (`.tar.gz`), max 50 MB compressed.
- Paths must be safe relative POSIX paths.
- Symlinks, hard links, absolute paths, parent traversal, and special entries are rejected.
- `poof-backend-artifact.json` must parse as JSON.
- `entrypoint` must point to a regular file inside the archive.
- Optional `apiSpecPath`, `queuesPath`, and `heartbeatPath` must point to regular files when present.

## Lifecycle Semantics

`poof deploy backend` creates a backend deploy checkpoint and deploys the bundle to the draft API URL. After a successful deploy:

- `poof project status -p <id> --json` reports `latestTask.backendBundleUrl`.
- The project's generation mode includes `backend-artifact`.
- Preview and production deploys preserve the active backend artifact instead of rebuilding source PartyServer code.
- Static frontend deploys preserve the active backend artifact.
- Backend artifact deploys preserve the active static frontend artifact.
- A later source-backed API/backend task supersedes the uploaded backend artifact.
- UI-only and policy-only tasks preserve the active uploaded backend.

For a full local lifecycle, validate both sides after each transition:

```bash
# Deploy backend, then smoke the API
poof deploy backend -p "$PROJECT_ID" --archive backend-worker.tar.gz
curl "$(poof project status -p "$PROJECT_ID" --json | jq -r '.connectionInfo.draft.backendUrl')/api/health"

# Deploy static UI, then re-smoke the API
poof deploy static -p "$PROJECT_ID" --archive dist.tar.gz
curl "$(poof project status -p "$PROJECT_ID" --json | jq -r '.connectionInfo.draft.backendUrl')/api/health"

# Promote and verify both UI and backend URLs
poof deploy preview -p "$PROJECT_ID"
poof deploy production -p "$PROJECT_ID" --yes
```

## Verification

Use `poof verify` to run lifecycle/API tests against the deployed backend. For projects with `backend-artifact` in generation mode, verification should not ask Poof's AI to regenerate backend source from the bundle.

Recommended checks after every backend artifact deploy:

- `poof project status -p <id> --json` shows the expected `backendBundleUrl`.
- Draft API URL responds to health and representative API routes.
- CORS preflight works for the draft UI origin if the frontend calls the API.
- If a static UI is active, the draft UI still serves the uploaded static marker after backend deploy.
- Before preview/production, run `poof security scan -p <id> --wait` or let `poof ship` run the scan.

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `archive is not a gzip-compressed tar file` | Archive is missing gzip wrapper | Use `tar czf backend-worker.tar.gz -C .poof-backend-bundle .` |
| `backend archive must include poof-backend-artifact.json` | Manifest missing from archive root | Add the manifest before archiving |
| `entrypoint ... was not found` | Manifest points at the wrong bundled file | Inspect `.poof-backend-bundle` and update `entrypoint` |
| Deploy serves old backend | Preview/production promoted before backend task completed | Re-check `poof task get <taskId>` and `poof project status` |
| Production deploy blocked | Missing security review, membership, or required secrets | Run `poof security scan --wait`, `poof deploy check`, and `poof secrets get --environment production` |
