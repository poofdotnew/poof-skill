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
| `poof build -m "..." [--mode MODE] [--public] [--stdin]`   | Create a project, wait for AI to finish. Returns project ID and URLs. Mode: `full` (default), `policy`, `ui,policy`, `backend,policy`. Bare values `ui` and `backend` are also accepted (policy is auto-included).                                                                                                                                                        |
| `poof iterate -p <id> -m "..." [--stdin] [--file img.png]` | Send a chat message, wait for AI to finish, show test results. Attach images with `--file`.                                                                                                                                                                                                                                                                               |
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
| `poof chat active -p <id>`                                   | Check if AI is currently processing.                                                                        |
| `poof chat cancel -p <id>`                                   | Cancel in-progress AI execution.                                                                            |
| `poof chat steer -p <id> -m "..." [--stdin]`                 | Redirect AI mid-execution without cancelling.                                                               |

### Files

| Command                                                 | Description                                                                                       |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `poof files get -p <id>`                                | Get all source files. **Requires a credit purchase.**                                             |
| `poof files update -p <id> --file PATH --content "..."` | Update a single file.                                                                             |
| `poof files update -p <id> --from-json FILE`            | Update multiple files from a JSON map.                                                            |
| `poof files upload -p <id> --file <path>`               | Upload an image to project storage. Returns CDN URL. Supported: PNG, JPEG, GIF, WebP (max 3.4MB). |

### Tasks & Testing

| Command                                                            | Description                                                                                                                                    |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `poof task list -p <id> [--change-id ID] [--limit N] [--offset N]` | List tasks (builds, deployments, downloads). Defaults to the 20 newest project tasks; use `--change-id latest` to narrow to the latest change. |
| `poof task get <taskId> -p <id>`                                   | Get task details.                                                                                                                              |
| `poof task test-results -p <id> [--limit N] [--offset N]`          | Structured policy + UI test results — pass/fail, errors, counts. Default limit: 100.                                                           |

### Deployment

| Command                                                                                                           | Description                                                                                                                                                                                                                           |
| ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `poof deploy check -p <id>`                                                                                       | Check publish eligibility (payment status, readiness).                                                                                                                                                                                |
| `poof deploy preview -p <id> [--dry-run] [--yes]`                                                                 | Deploy to mainnet preview. Optional: `--allowed-addresses addr1,addr2` (max 10), `--constants-overrides '{...}'`, `--config '{...}'`.                                                                                                 |
| `poof deploy production -p <id> --yes [--dry-run]`                                                                | Deploy to production. Optional: `--constants-overrides '{...}'`, `--config '{...}'`.                                                                                                                                                  |
| `poof deploy mobile -p <id> --platform <platform> --app-name "<name>" --app-icon-url "<url>" [--dry-run] [--yes]` | Publish mobile app. All three flags are required. Platform: `seeker`, `ios`, `android`. Optional: `--app-description`, `--theme-color` (default: `#0a0a0a`), `--draft`, `--target-environment` (`draft`, `mainnet-preview`), `--yes`. |
| `poof deploy static -p <id> --archive dist.tar.gz [--title T] [--description D] [--dry-run]`                      | Deploy a pre-built static frontend. Archive must be a gzip-compressed tar (`.tar.gz`).                                                                                                                                                |
| `poof deploy download -p <id>`                                                                                    | Start code export. Returns task ID.                                                                                                                                                                                                   |
| `poof deploy download-url -p <id> --task <taskId>`                                                                | Get signed download URL (5-min expiry).                                                                                                                                                                                               |

### Security

| Command                                        | Description                                                                                        |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `poof security scan -p <id> [--task <taskId>]` | Run security audit on policies, code, and config. Use `--task` to check status of a previous scan. |

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
| `poof completion [bash\|zsh\|fish\|powershell]` | Generate shell completion script for the specified shell.                                |

## Global Flags

All commands support these flags:

| Flag                  | Description                                                     |
| --------------------- | --------------------------------------------------------------- |
| `-p, --project <id>`  | Project ID (or set `default_project_id` via `poof config set`). |
| `--env <environment>` | Target environment: `production` (default), `staging`, `local`. |
| `--json`              | Output as JSON (for scripting and parsing).                     |
| `--quiet`             | Minimal output (IDs and URLs only).                             |

## JSON Output Shapes

When using `--json`, commands return structured JSON. Key shapes:

| Command                  | JSON Shape                                                                                                                                                                                                                           |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `poof credits balance`   | `{ credits: { daily: { remaining, allotted, resetsAt }, subscription: { remaining, purchased }, addOn: { remaining, purchased }, total } }` (Note: `subscription` is deprecated but still present. Credit values may be fractional.) |
| `poof project messages`  | `{ messages: [{ id, role, content, createdAt, status }], hasMore }`                                                                                                                                                                  |
| `poof chat active`       | `{ active: boolean, status: "ok" \| "error" }`                                                                                                                                                                                       |
| `poof project status`    | `{ project: {...}, latestTask: {...}, publishState: {...}, urls: { draft, mainnetPreview, preview, production }, connectionInfo: {...} }`                                                                                            |
| `poof build`             | `{ projectId, urls: {...}, project: {...} }`                                                                                                                                                                                         |
| `poof iterate`           | `{ results: [{id, source, fileName, testName, status, counts: {steps, expects, failed}, lastError, duration, startedAt}], summary: {total, passed, failed, errors, running}, hasMore }`                                              |
| `poof task test-results` | `{ results: [{id, source, fileName, testName, status, counts: {steps, expects, failed}, lastError, duration, startedAt}], summary: {total, passed, failed, errors, running}, hasMore }`                                              |
| `poof files get`         | `{ files: { [path]: content } }`                                                                                                                                                                                                     |

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
