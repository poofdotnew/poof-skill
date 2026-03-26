# CLI Command Reference

## Contents
- [Workflow Commands](#workflow-commands)
- [All Commands](#all-commands)
- [JSON Output Shapes](#json-output-shapes)
- [Error Handling](#error-handling)

## Workflow Commands

These composite commands handle multi-step operations automatically (polling, security checks, etc.) and are the **primary interface** for AI agents:

| Command | What it does |
|---------|-------------|
| `poof build -m "..." [--mode MODE]` | Create a project, wait for AI to finish. Returns project ID and URLs. |
| `poof iterate -p <id> -m "..."` | Send a chat message, wait for AI to finish, show test results. |
| `poof ship -p <id> [--target TARGET] [--signed-permit <tx>]` | Run security scan, check eligibility, deploy. Targets: `preview` (default), `production`, `mobile`. `--signed-permit` is required when target is `preview`. |

## All Commands

### Authentication

| Command | Description |
|---------|-------------|
| `poof keygen` | Generate a new Solana keypair. Outputs `SOLANA_PRIVATE_KEY` and `SOLANA_WALLET_ADDRESS`. |
| `poof auth login` | Authenticate and cache token. |
| `poof auth status` | Show current auth status and token validity. |
| `poof auth logout` | Clear cached credentials. |

### Project Management

| Command | Description |
|---------|-------------|
| `poof project list [--limit N] [--offset N]` | List all projects. |
| `poof project create -m "..." [--mode MODE] [--public]` | Create a project (AI starts building). Does NOT wait — use `poof build` instead. |
| `poof project status -p <id>` | Get metadata, task status, deployment URLs, connection info. |
| `poof project update -p <id> [--title T] [--description D] [--slug S]` | Update project metadata. |
| `poof project delete -p <id> --yes` | Delete a project permanently. |
| `poof project messages -p <id>` | Get conversation history. |

### Chat & AI Control

| Command | Description |
|---------|-------------|
| `poof chat send -p <id> -m "..."` | Send a message (does NOT wait). Use `poof iterate` instead for wait + results. |
| `poof chat active -p <id>` | Check if AI is currently processing. |
| `poof chat cancel -p <id>` | Cancel in-progress AI execution. |
| `poof chat steer -p <id> -m "..."` | Redirect AI mid-execution without cancelling. |

### Files

| Command | Description |
|---------|-------------|
| `poof files get -p <id>` | Get all source files. **Requires a credit purchase.** |
| `poof files update -p <id> --file PATH --content "..."` | Update a single file. |
| `poof files update -p <id> --from-json FILE` | Update multiple files from a JSON map. |

### Tasks & Testing

| Command | Description |
|---------|-------------|
| `poof task list -p <id>` | List tasks (builds, deployments, downloads). |
| `poof task get <taskId> -p <id>` | Get task details. |
| `poof task test-results -p <id>` | Structured test results — pass/fail, errors, counts. |

### Deployment

| Command | Description |
|---------|-------------|
| `poof deploy check -p <id>` | Check publish eligibility (payment status, readiness). |
| `poof deploy preview -p <id> --signed-permit <tx>` | Deploy to mainnet preview. `--signed-permit` is required. |
| `poof deploy production -p <id> --yes` | Deploy to production. |
| `poof deploy mobile -p <id> --platform <platform> --app-name "<name>" --app-icon-url "<url>"` | Publish mobile app. All three flags are required. Platform: `seeker`, `ios`, `android`. |
| `poof deploy download -p <id>` | Start code export. Returns task ID. |
| `poof deploy download-url -p <id> --task <taskId>` | Get signed download URL (5-min expiry). |

### Security

| Command | Description |
|---------|-------------|
| `poof security scan -p <id>` | Run security audit on policies, code, and config. |

### Secrets

| Command | Description |
|---------|-------------|
| `poof secrets get -p <id>` | List required and optional secret names. |
| `poof secrets set -p <id> KEY=VALUE [KEY=VALUE...]` | Set secret values. |

### Domains

| Command | Description |
|---------|-------------|
| `poof domain list -p <id>` | List custom domains. **Requires a credit purchase.** |
| `poof domain add DOMAIN -p <id> [--default]` | Add a custom domain. **Requires a credit purchase.** |

### Logs

| Command | Description |
|---------|-------------|
| `poof logs -p <id> [--environment ENV] [--limit N]` | Get runtime logs. |

### Credits & Payments

| Command | Description |
|---------|-------------|
| `poof credits balance` | Credit balance: daily, add-on, totals. |
| `poof credits topup [--quantity N]` | Buy credits via x402 USDC. Also unlocks paid features. |

### Templates

| Command | Description |
|---------|-------------|
| `poof template list [--category CAT] [--search Q] [--sort SORT] [--limit N] [--skip N]` | Browse available project templates. `--sort` values: `newest`, `most_used`, `staff_picks`. |

### AI Preferences

| Command | Description |
|---------|-------------|
| `poof preferences get` | View current AI model tier preferences. |
| `poof preferences set KEY=VALUE [KEY=VALUE...]` | Set tiers: `average`, `smart`, `genius`. Requires a credit purchase. |

### Configuration

| Command | Description |
|---------|-------------|
| `poof config show` | Display current CLI configuration. |
| `poof config set KEY VALUE` | Set persistent config (e.g., `default_project_id`, `environment`). |
| `poof browser` | Generate a sign-in link for poof.new using CLI credentials. |

## Global Flags

All commands support these flags:

| Flag | Description |
|------|-------------|
| `-p, --project <id>` | Project ID (or set `default_project_id` via `poof config set`). |
| `--env <environment>` | Target environment: `production` (default), `staging`, `local`. |
| `--json` | Output as JSON (for scripting and parsing). |
| `--quiet` | Minimal output (IDs and URLs only). |

## JSON Output Shapes

When using `--json`, commands return structured JSON. Key shapes:

| Command | JSON Shape |
|---------|-----------|
| `poof credits balance` | `{ credits: { daily: { remaining, allotted, resetsAt }, subscription: { remaining, purchased }, addOn: { remaining, purchased }, total } }` (Note: `subscription` is deprecated but still present in the response.) |
| `poof project messages` | `{ messages: [{ id, role, content, createdAt, status }] }` |
| `poof chat active` | `{ active: boolean, status: "ok" \| "error" }` |
| `poof project status` | `{ project: {...}, latestTask: {...}, publishState: {...}, urls: { preview, production, draft }, connectionInfo: {...} }` |
| `poof build` | `{ projectId, urls: {...} }` |
| `poof iterate` | `{ results: [{id, fileName, testName, status, counts: {steps, expects, failed}, lastError, duration, startedAt}], summary: {total, passed, failed, errors, running}, hasMore }` (If no tests exist, may return a success message with no structured test data.) |
| `poof task test-results` | `{ results: [{id, fileName, testName, status, counts, lastError}], summary: {total, passed, failed, errors} }` |
| `poof files get` | `{ files: { [path]: content } }` |

## Error Handling

The CLI provides context-aware error messages:

| Error | Cause | Fix |
|-------|-------|-----|
| `Not authenticated` | Invalid/expired token | Run `poof auth login` |
| `Project not found` | Wrong ID or not owner | Check with `poof project list` |
| `You have run out of credits` | No credits remaining | Run `poof credits topup` or wait for daily reset |
| `Credit purchase required` | Paid feature without payment | Run `poof credits topup` to unlock |

### Tips

- The CLI automatically refreshes expired tokens
- All IDs (projectId, messageId) are generated automatically by the CLI
- `poof credits balance` is free — check before long workflows
