# CLI Command Reference

## Contents

- [Workflow Commands](#workflow-commands)
- [All Commands](#all-commands)
- [JSON Output Shapes](#json-output-shapes)
- [Error Handling](#error-handling)

## Workflow Commands

These composite commands handle multi-step operations automatically (polling, security checks, etc.) and are the **primary interface** for AI agents:

| Command                                                    | What it does                                                                                                                                                                                                                                                                                                                                                              |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `poof build -m "..." [--mode MODE] [--public] [--stdin] [--file img.png]`   | Create a project, wait for AI to finish. Returns project ID and URLs. Mode: `full` (default), `policy`, `ui,policy`, `backend,policy`. Bare values `ui` and `backend` are also accepted (policy is auto-included). Attach reference images (PNG, JPEG, GIF, WebP; ≤3.4MB each) with `--file` — the CLI pre-uploads each image to Poof's global storage and embeds the URLs in the initial firstMessage so the AI sees text + images together. |
| `poof iterate -p <id> -m "..." [--stdin] [--file img.png]` | Send a chat message, wait for AI to finish, show test results. Attach images with `--file`. JSON output includes both `summary` (latest per file) and `freshSummary` (results created during this turn); text output uses freshSummary and says "no tests ran during this turn" when nothing new was executed. For strict pass/fail gates use `poof verify` instead. |
| `poof verify -p <id> [--ui-tests auto\|true\|false] [--skip-url-probe] [-m "..."]` | Send canonical verification prompt, wait for AI, validate fresh test results. Auto-detects backend-only/static-deploy projects and skips UI tests. Exits non-zero if no fresh results or any failure. Use `--ui-tests=false` to force lifecycle-only. Use `--ui-tests=true` only for Poof-generated UIs or for static deploys where source-authored `lifecycle-actions/ui-test-*.json` files were uploaded first; pair it with `-m` to say "run existing tests, do not generate from dist." |
| `poof ship -p <id> [-t TARGET] [--dry-run] [--yes]`        | Run security scan, check eligibility, deploy. Targets: `preview` (default), `production`, `mobile`. Passes through all target-specific flags (e.g., `--allowed-addresses`, `--constants-overrides`, `--config` for preview/production; `--platform`, `--app-name`, `--app-icon-url`, `--app-description`, `--theme-color`, `--draft`, `--target-environment` for mobile). |

## All Commands

### Authentication

| Command            | Description                                                                              |
| ------------------ | ---------------------------------------------------------------------------------------- |
| `poof keygen`      | Generate a new Solana keypair. Outputs `SOLANA_PRIVATE_KEY` and `SOLANA_WALLET_ADDRESS`. |
| `poof auth login`  | Authenticate and cache token.                                                            |
| `poof auth status` | Show current auth status and token validity.                                             |
| `poof auth logout` | Clear cached credentials.                                                                |

### Project Management

| Command                                                                                                                    | Description                                                                                                                                                                                                                   |
| -------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `poof project list [--limit N] [--offset N]`                                                                               | List all projects. Default limit: 10.                                                                                                                                                                                         |
| `poof project create -m "..." [--mode MODE] [--public] [--stdin]`                                                          | Create a project (AI starts building). Does NOT wait — use `poof build` instead. Mode: `full` (default), `policy`, `ui,policy`, `backend,policy`. Bare values `ui` and `backend` are also accepted (policy is auto-included). |
| `poof project status -p <id>`                                                                                              | Get metadata, task status, deployment URLs, connection info.                                                                                                                                                                  |
| `poof project update -p <id> [--title T] [--description D] [--slug S] [--public] [--generation-mode MODE] [--network NET]` | Update project settings.                                                                                                                                                                                                      |
| `poof project delete -p <id> --yes [--dry-run]`                                                                            | Delete a project permanently. Use `--dry-run` to preview without deleting.                                                                                                                                                    |
| `poof project messages -p <id> [--limit N] [--offset N]`                                                                   | Get conversation history. Default limit: 50.                                                                                                                                                                                  |

### Chat & AI Control

| Command                                                      | Description                                                                                                 |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| `poof chat send -p <id> -m "..." [--stdin] [--file img.png]` | Send a message (does NOT wait). Use `poof iterate` instead for wait + results. Attach images with `--file`. |
| `poof chat active -p <id>`                                   | Check if AI is currently processing. JSON output includes `state` (`running`, `queued`, or `idle`).        |
| `poof chat cancel -p <id>`                                   | Cancel in-progress AI execution.                                                                            |
| `poof chat steer -p <id> -m "..." [--stdin]`                 | Redirect AI mid-execution without cancelling.                                                               |

### Files

| Command                                                 | Description                                                                                       |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `poof files get -p <id> [--list\|--stat] [--path GLOB]` | Get source files. **Requires a credit purchase.** Use `--list` to print paths only, `--stat` for paths + byte counts, or `--path` to filter by glob (e.g. `"src/**/*.tsx"`, `"HomePage.tsx"`, `"lifecycle-actions/*.json"`). With `--path` set, text mode prints the matched file contents with `==> path <==` headers; without it, text mode lists paths and hints at `--json` for a full dump. Prefer the filtered forms to keep output small. |
| `poof files update -p <id> --file PATH --content "..."` | Update a single file.                                                                             |
| `poof files update -p <id> --from-json FILE`            | Update multiple files from a JSON map of project paths to file contents. Use paths like `lifecycle-actions/ui-test-create-post.json` for source-authored UI lifecycle tests. |
| `poof files upload -p <id> --file <path>`               | Upload an image to project storage. Returns CDN URL. Supported: PNG, JPEG, GIF, WebP (max 3.4MB). |

### Tasks & Testing

| Command                                                            | Description                                                                                                                                    |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `poof task list -p <id> [--change-id ID] [--limit N] [--offset N]` | List tasks (builds, deployments, downloads). Defaults to the 20 newest project tasks; use `--change-id latest` to narrow to the latest change. |
| `poof task get <taskId> -p <id>`                                   | Get task details.                                                                                                                              |
| `poof task test-results -p <id> [--limit N] [--offset N] [--history]` | Structured policy + UI test results — pass/fail, errors, counts. Default limit: 100. Results are collapsed to latest run per test file by default; pass `--history` to see the full raw history instead. |

### Deployment

| Command                                                                                                           | Description                                                                                                                                                                                                                           |
| ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `poof deploy check -p <id>`                                                                                       | Check publish eligibility (payment status, readiness).                                                                                                                                                                                |
| `poof deploy preview -p <id> [--dry-run] [--yes]`                                                                 | Deploy to mainnet preview and wait for deploy task to complete. Optional: `--allowed-addresses addr1,addr2` (max 10), `--constants-overrides '{...}'`, `--config '{...}'`.                                                             |
| `poof deploy production -p <id> --yes [--dry-run]`                                                                | Deploy to production and wait for deploy task to complete. Optional: `--constants-overrides '{...}'`, `--config '{...}'`.                                                                                                              |
| `poof deploy mobile -p <id> --platform <platform> --app-name "<name>" --app-icon-url "<url>" [--dry-run] [--yes]` | Publish mobile app. All three flags are required. Platform: `seeker`, `ios`, `android`. Optional: `--app-description`, `--theme-color` (default: `#0a0a0a`), `--draft`, `--target-environment` (`draft`, `mainnet-preview`), `--yes`. |
| `poof deploy static -p <id> --archive dist.tar.gz [--title T] [--description D] [--dry-run]`                      | Deploy a pre-built static frontend. Archive must be a gzip-compressed tar (`.tar.gz`).                                                                                                                                                |
| `poof deploy download -p <id>`                                                                                    | Start code export. Returns task ID.                                                                                                                                                                                                   |
| `poof deploy download-url -p <id> --task <taskId>`                                                                | Get signed download URL (5-min expiry).                                                                                                                                                                                               |

### Data (runtime reads/writes)

Agent-facing runtime operations against a project's data plane — all CLI-driven. The CLI handles both the offchain submit path and real Solana tx signing + submit (mainnet) under the hood.

**Two targeting modes** (every command below accepts either, but not both):

- **Project-based** — `-p <project-id> [-e draft|preview|production]`. Default `-e` is `draft`. Looks up the right Tarobase appId from the project's `connectionInfo`. Requires project access.
- **Shared-appid** — `--app-id <appId> --chain offchain|mainnet`. Talks to a Tarobase appId directly, e.g. a shared primitives library someone else has deployed. Auth is still your own wallet; no project access needed. User-scoped paths (`user/$userId/...`) keep callers sandboxed to their own data within the shared appId.

Use `poof data app-ids -p <id>` to list a project's appIds per environment so you have the right `--app-id` / `--chain` pair for shared-appid mode.

| Command                                                                                       | Description                                                                                                                                            |
| --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `poof data set <TARGET> --path <path> --data '<json>'`                                        | Write a single document. Path is the full Poof collection path (e.g. `memories/<addr>` or `user/<addr>/TokenTransfer/<id>`).                          |
| `poof data set-many <TARGET> --from-json <file>`                                              | Atomic multi-doc write. Payload is either a `[{path, document}, ...]` array or `{"documents":[...]}`. Any rule/hook failure rolls back the whole bundle. |
| `poof data get <TARGET> --path <path>`                                                        | Read one document or list a collection by path.                                                                                                        |
| `poof data get-many <TARGET> --from-json <paths.json>`                                        | Batch-read paths from a JSON string array; returns one result per path in order.                                                                       |
| `poof data query <TARGET> --name <queryName> [--args '<json>'] [--path <path>]`               | Run a policy query. Default path is `queries/<name>`. Use `--path` for queries attached to a specific collection — e.g. `--path "user/<addr>/BalanceCheck/any" --name simulate` for a guard's simulate query. Inside queries, named `queryArgs` resolve as `@newData.<field>`. |
| `poof data app-ids -p <id>`                                                                   | List a project's Tarobase appIds per env (draft/preview/production) plus the chain each maps to. Read this once; reuse the appId in shared-appid mode. |

`<TARGET>` above is one of: `-p <project-id> [-e draft|preview|production]` or `--app-id <id> --chain offchain|mainnet`.

### Security

| Command                                        | Description                                                                                        |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `poof security scan -p <id> [--wait] [--task <taskId>]` | Run security audit on policies, code, and config. Returns immediately with the scan ID by default; pass `--wait` to block until the scan finishes and surface critical/high/medium/low finding counts inline. Use `--task` to target a specific previous task. |

### Diagnostics & Verification

| Command                                                                             | Description                                                                                                                                                                                                                                      |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `poof verify -p <id> [--ui-tests auto\|true\|false] [--skip-url-probe] [-m "..."]` | Canonical post-build test gate. Snapshots existing tests, sends verification prompt, waits for AI, only counts fresh results. Exits non-zero if no fresh results or any failure. Auto-detects backend-only/static-deploy projects and skips UI tests unless forced. For static deploys, force UI tests only to run existing source-authored `ui-test-*.json` files. |
| `poof doctor -p <id> [--skip-url-probe] [--task-limit N]`                            | Aggregated health check: project status, deploy flags, AI active state, recent tasks, test summary (latest per file), draft URL probe. `--task-limit` controls how many recent tasks to include (default 10). Outputs a verdict (`healthy`, `ai_running`, `deployed_with_test_failures`, `deployed_without_tests`, `deploy_pending`, `incomplete`, `unknown`) and next-step hint. |

### Secrets

| Command                                             | Description                                                                                                     |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `poof secrets get -p <id> [--environment ENV]`      | List required and optional secret names. Environment: `development` (default), `mainnet-preview`, `production`. |
| `poof secrets set -p <id> KEY=VALUE [KEY=VALUE...]` | Set secret values. Secrets are stored globally and available across all environments.                           |

### Domains

| Command                                      | Description                                          |
| -------------------------------------------- | ---------------------------------------------------- |
| `poof domain list -p <id>`                   | List custom domains. **Requires a credit purchase.** |
| `poof domain add DOMAIN -p <id> [--default]` | Add a custom domain. **Requires a credit purchase.** |

### Logs

| Command                                                          | Description                                                                                                   |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `poof logs -p <id> [--environment ENV] [--limit N] [--offset N]` | Get runtime logs. Environment: `development`, `mainnet-preview`, `production`. Limit range: 1-50, default 50. |

### Credits & Payments

| Command                             | Description                                                                                     |
| ----------------------------------- | ----------------------------------------------------------------------------------------------- |
| `poof credits balance`              | Credit balance: daily, add-on, totals.                                                          |
| `poof credits topup [--quantity N]` | Buy credits via x402 USDC (1-10 packages, each = 50 credits / $15). Also unlocks paid features. |

### Templates

| Command                                                                                 | Description                                                                                |
| --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `poof template list [--category CAT] [--search Q] [--sort SORT] [--limit N] [--skip N]` | Browse available project templates. `--sort` values: `newest`, `most_used`, `staff_picks`. |

### AI Preferences

| Command                                         | Description                                                          |
| ----------------------------------------------- | -------------------------------------------------------------------- |
| `poof preferences get`                          | View current AI model tier preferences.                              |
| `poof preferences set KEY=VALUE [KEY=VALUE...]` | Set tiers: `average`, `smart`, `genius`. Requires a credit purchase. |

### Configuration

| Command                                         | Description                                                                              |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `poof config show`                              | Display current CLI configuration.                                                       |
| `poof config set KEY VALUE`                     | Set persistent config. Valid keys: `default_project_id`, `environment`, `output_format`. |
| `poof browser`                                  | Generate a sign-in link for poof.new using CLI credentials.                              |
| `poof version`                                  | Print version, commit, and build date.                                                   |
| `poof update`                                   | Update poof to the latest GitHub release. Homebrew-managed installs should use `brew upgrade poofdotnew/tap/poof`. |
| `poof update --check`                           | Check the latest release and platform asset without installing.                          |
| `poof update --force`                           | Install the latest release even when the current version cannot be compared, such as a `dev` build. |

## Debugging below the CLI

You shouldn't need to talk to Poof's data-plane HTTP endpoints directly — `poof data` is the interface. If you do end up in the raw-HTTP layer (e.g. to reproduce a report that hit the REST endpoint), the two non-obvious points the CLI hides for you:

- **Use the Cognito `idToken` as the Bearer token, not `accessToken`.** The data-plane API verifies against the ID token — `accessToken` returns `401 "Error verifying auth token"`.
- **Scope auth to the per-env app ID, not the top-level poof.new app.** Nonce + signed-session have to be issued against the per-environment app ID from `poof project status --json` (`connectionInfo.<env>.tarobaseAppId` — a legacy field name; treat it as your Poof app ID for that environment). A session scoped to the wrong app returns `401` on every call.

Everything else — `PUT /items`, `GET /items`, `POST /queries`, offchain submit, LUT-aware VersionedTransaction assembly for mainnet — is baked into the CLI and exposed through `poof data`.

## Global Flags

All commands support these flags:

| Flag                  | Description                                                     |
| --------------------- | --------------------------------------------------------------- |
| `-p, --project <id>`  | Project ID (or set `default_project_id` via `poof config set`). |
| `--env <environment>` | Target environment: `production` (default), `staging`, `local`. |
| `--json`              | Output as JSON (for scripting and parsing).                     |
| `--quiet`             | Minimal output (IDs and URLs only).                             |
| `--no-update-check`   | Skip automatic update checks and update-available notices.      |

## JSON Output Shapes

When using `--json`, commands return structured JSON. Key shapes:

| Command                  | JSON Shape                                                                                                                                                                                                                           |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `poof credits balance`   | `{ credits: { daily: { remaining, allotted, resetsAt }, subscription: { remaining, purchased }, addOn: { remaining, purchased }, total } }` (Note: `subscription` is deprecated but still present. Credit values may be fractional.) |
| `poof project messages`  | `{ messages: [{ id, role, content, createdAt, status }], hasMore }`                                                                                                                                                                  |
| `poof chat active`       | `{ active: boolean, state: "running" \| "queued" \| "idle", status: "ok" \| "error" }`                                                                                                                                             |
| `poof project status`    | `{ project: {...}, latestTask: {...}, publishState: { draft: { deployed, url }, preview: {...}, live: {...}, mobile: {...} }, urls: { draft, mainnetPreview, preview, production }, connectionInfo: {...} }`                      |
| `poof build`             | `{ projectId, urls: {...}, project: {...}, publishState: {...}, draftReady: boolean }`                                                                                                                                                |
| `poof iterate`           | `{ results: [...], summary: {total, passed, failed, errors, running}, freshResults: [...], freshSummary: {total, passed, failed, errors, running}, hasMore }` — `freshResults`/`freshSummary` cover only results created during this turn; `results`/`summary` are latest-per-file across all history. |
| `poof verify`            | `{ projectId, passed: boolean, freshSummary: {total, passed, failed, errors, running}, freshResults: [...], baselineSize: number, probe: { url, statusCode, reachable, draftDeployedFlag } }` — `passed` is true only when freshSummary.total > 0 AND no failures/errors. |
| `poof doctor`            | `{ projectId, project: {...}, urls: {...}, publishState: {...}, draftDeployedFlag, previewDeployedFlag, liveDeployedFlag, ai: { active, state, status }, recentTasks: [...], testSummary: {...}, rawTestSummary: {...}, probe: {...}, verdict: string, errors: [...] }` |
| `poof task test-results` | `{ results: [{id, source, fileName, testName, status, counts: {steps, expects, failed}, lastError, duration, startedAt}], summary: {total, passed, failed, errors, running}, hasMore }`                                              |
| `poof files get`         | Default: `{ files: { [path]: content } }`. With `--list`: `{ files: ["path1", ...], total: N }`. With `--stat`: `{ files: [{ path, bytes }], total: N, bytes: N }`. |

## Error Handling

The CLI provides context-aware error messages:

| Error                              | Cause                        | Fix                                              |
| ---------------------------------- | ---------------------------- | ------------------------------------------------ |
| `Not authenticated`                | Invalid/expired token        | Run `poof auth login`                            |
| `Project not found`                | Wrong ID or not owner        | Check with `poof project list`                   |
| `You have run out of free credits` | No credits remaining         | Run `poof credits topup` or wait for daily reset |
| `Credit purchase required`         | Paid feature without payment | Run `poof credits topup` to unlock               |

### Tips

- The CLI automatically refreshes expired tokens
- All IDs (projectId, messageId) are generated automatically by the CLI
- `poof credits balance` is free — check before long workflows
