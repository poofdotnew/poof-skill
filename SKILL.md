---
name: poof
description: >-
  Use when building AI agents that interact with Poof (poof.new), creating or
  managing Solana dApps programmatically, using the poof CLI, or connecting a
  local frontend to a Poof backend. Triggers: "poof project", "deploy dApp",
  "create poof app", "poof.new", "poof CLI", "poof build", "poof-cli",
  "Solana agent", "blockchain app builder", "@pooflabs/web".
---

# Poof CLI

Build autonomous AI agents that create, deploy, and manage full-stack Solana applications on [poof.new](https://poof.new) using the `poof` CLI — no custom code needed.

## How It Works

The `poof` CLI wraps authentication, API calls, and polling into simple commands. Your agent runs shell commands and parses output.

```
Your Agent ──► poof CLI (auth + API + polling built in) ──► poof.new
```

## Timeouts for Long-Running Commands

**Critical:** `poof build`, `poof iterate`, `poof verify`, and `poof ship` poll until the Poof AI finishes, which can take **5–15+ minutes**. You **must** set an extended timeout or run these commands in the background, or they will be killed mid-execution.

| Command                                 | Typical duration | Recommended approach                  |
| --------------------------------------- | ---------------- | ------------------------------------- |
| `poof build`                            | 3–10 min         | `run_in_background: true`             |
| `poof iterate`                          | 2–10 min         | `run_in_background: true`             |
| `poof verify`                           | 5–15 min         | `run_in_background: true`             |
| `poof ship`                             | 1–5 min          | `run_in_background: true`             |
| `poof security scan`                    | 1–3 min          | `timeout: 600000`                     |
| `poof deploy preview/production/mobile` | 1–3 min          | `timeout: 600000`                     |
| `poof deploy static`                    | 30s–2 min        | `timeout: 600000`                     |
| `poof credits topup`                    | 30–90 sec        | `timeout: 120000` (default)           |
| `poof files get`                        | 10–60 sec        | `timeout: 120000` (default)           |
| All other commands                      | < 30 sec         | `timeout: 120000` (default)           |

In Claude Code, the Bash tool's max `timeout` is 600000ms (10 min). For commands that can exceed 10 minutes (`poof build`, `poof iterate`, `poof verify`, `poof ship`), **always use `run_in_background: true`** — you'll be notified when the command completes. For shorter commands, set `timeout` directly. In other harnesses, use whatever mechanism your tool runner provides to extend the execution timeout or run commands asynchronously.

**Internal poll deadline:** The CLI itself enforces a 30-minute poll deadline on build/iterate/verify, so a legitimate verify run that generates tests, fixes failures, and reruns is no longer cut off at 10 minutes.

**Important:** `poof build` now waits for the initial AI turn plus draft deploy health, but it still does **not** prove lifecycle or UI tests passed. Use `poof verify -p <id>` for the canonical post-build verification flow. `poof iterate` is still a general chat command, so test-generation prompts should be followed by `poof verify` or `poof task test-results --json` if you need strict programmatic pass/fail behavior. `iterate` now reports **fresh** test counts (results created during this turn) and will say "no tests ran during this turn" instead of falsely claiming an older suite passed — but the strict gate is still `verify`.

**Important:** `poof build` success text is not the same as a healthy draft deploy. After build, run `poof project status -p <id> --json` and confirm the canonical project plus `publishState.draft.deployed`. If you need draft UI evidence, probe the advertised draft URL too. Treat HTTP `404` as missing deploy evidence, and treat `publishState.draft.deployed=false` plus a non-`404`/`200` smoke probe as inconsistent evidence that still needs explicit logging and follow-up before you call the build QA-ready.

**Shell safety:** For long or multi-line prompts, prefer a quoted temp file over one giant inline `-m "..."` command. `--stdin` is now strict, so only use it with a real pipe such as `cat prompt.txt | poof iterate --stdin`. In Paperclip/Codex heartbeat runners and other non-interactive tool environments, prefer a temp-file-backed prompt or a compact shell-safe `-m` string instead of assuming stdin will stay open. If you capture an exit code in zsh after `poof build` / `poof iterate`, use `rc=$?`, not `status=$?`.

**Claude Code example:**

```
# For commands that may exceed 10 min: run in background
Bash tool: command="poof build -m '...'", run_in_background=true

# For commands ≤10 min: set timeout
Bash tool: command="poof security scan -p <id>", timeout=600000
```

## Authentication

### Keypair Setup

Generate a Solana keypair (or use one the user provides):

```bash
# Generate a new keypair and append to .env
poof keygen >> .env

# .env now contains:
# SOLANA_PRIVATE_KEY=<base58-private-key>
# SOLANA_WALLET_ADDRESS=<public-address>
```

Then authenticate:

```bash
poof auth login
```

> Generate a fresh keypair for each agent. Only use an existing keypair if the user explicitly provides one.

| Variable                | Purpose                                                           | Required |
| ----------------------- | ----------------------------------------------------------------- | -------- |
| `SOLANA_PRIVATE_KEY`    | Solana private key (base58)                                       | Yes      |
| `SOLANA_WALLET_ADDRESS` | Your Solana wallet public address                                 | Yes      |
| `POOF_ENV`              | Target environment: `production` (default), `staging`, or `local` | No       |

To target staging: `poof --env staging` or set `POOF_ENV=staging` in `.env`.

> **Important:** The keypair you use becomes the project owner. The wallet address is set as `ADMIN_ADDRESS` in your project's constants, controlling admin permissions in policies.

## Quick Workflow

```bash
# 1. Setup (one-time)
poof keygen >> .env
poof auth login

# 2. Build (creates project, waits for AI to finish)
# generationMode: full (default) | policy | ui,policy | backend,policy
# Bare values "ui" and "backend" are also accepted (policy is auto-included)
poof build -m "Build a token-gated voting app" --mode full

# 3. Iterate (sends chat, waits for AI, shows test results)
poof iterate -p <project-id> -m "Add a leaderboard page"

# Safer pattern for a long prompt in heartbeat/non-interactive runners
PROMPT_FILE="$(mktemp)"
cat >"$PROMPT_FILE" <<'EOF'
Add wallet auth, gated posting, and lifecycle/UI tests.
Keep the prompt text shell-safe by storing it in a temp file first.
EOF
PROMPT="$(cat "$PROMPT_FILE")"
poof iterate -p <project-id> -m "$PROMPT"
rm -f "$PROMPT_FILE"

# 4. Steer mid-execution (optional)
poof chat steer -p <project-id> -m "Focus on the backend first"

# 5. Cancel if stuck
poof chat cancel -p <project-id>

# 6. Read/modify files directly (requires credit purchase)
poof files get -p <project-id>
poof files update -p <project-id> --file src/config.ts --content 'export const MAX = 100;'

# 7. Send a message with an image (e.g., UI screenshot for reference)
poof iterate -p <project-id> -m "Build a UI that looks like this" --file screenshot.png
poof files upload -p <project-id> --file logo.png   # standalone upload, returns CDN URL

# 8. Deploy (runs security scan + eligibility check + publish)
poof ship -p <project-id>
```

**Environment note:** After `poof build`, the intended runtime target is **Draft (Poofnet)** — a simulated blockchain with fake tokens. Do not assume the draft app is already reachable until `poof project status -p <id> --json` shows draft deploy state consistent with a live endpoint, or a direct draft URL probe returns non-`404`. To test with real mainnet tokens, deploy to **Preview** using `poof ship -p <id> --target preview`. See [docs/deployment.md](docs/deployment.md) for the full environment breakdown.

### Agent Workflow Checklist

Copy this checklist and track your progress:

```
- [ ] Setup: poof keygen >> .env && poof auth login
- [ ] Build: poof build -m "..." [--mode full|policy|backend,policy|ui,policy]
- [ ] Status: poof project status -p <id> --json   # confirm canonical id + draft deploy
- [ ] Verify: poof verify -p <id>                  # canonical strict policy + UI test gate
- [ ] Diagnose: poof doctor -p <id>                # only if verify fails — points at the next step
- [ ] Fix: poof iterate -p <id> -m "Fix: <error details>" && poof verify -p <id>
- [ ] Deploy: poof ship -p <id>
```

`poof verify` is the only test command an agent should rely on for pass/fail. It snapshots
existing test result IDs, sends the canonical lifecycle + UI verification prompt, then only
counts results created during that run. It exits non-zero if no fresh results appear or if
any fresh result failed, so a successful exit code is real evidence that tests ran and passed.
`poof iterate` is still a general chat command — only fall back to it for free-form fixes.

### Reality Checks

- If `poof project list --json` shows multiple similarly named projects and you do not already have a canonical project id from the user or repo artifacts, do not guess by title. Create or explicitly reconcile the canonical project first, then write that id into your artifact store before more `iterate` or deploy work.
- After `poof build`, always run `poof project status -p <id> --json`. Record the project id, URLs, and deploy state before you start retries or testing so later wakes do not guess which project is canonical.
- If `poof task test-results -p <id> --json` reports `summary.total == 0`, inspect `poof task list -p <id> --json`, `poof chat active -p <id> --json`, `poof logs -p <id>`, and `poof project messages -p <id> --limit 100 --json` before you assume tests passed or failed.
- `poof task test-results`, `poof iterate`, `poof verify`, and `poof doctor` now collapse test results to the **latest run per test file** by default. If you need to see the full history (e.g. to debug why an earlier run failed), use `poof task test-results -p <id> --history`.
- If `poof chat active -p <id> --json` stays `true` while task list shows no new work and logs show no recent activity, cancel that stale chat once with `poof chat cancel -p <id>` before the single allowed targeted retry.
- If the only visible tasks are still bootstrap/constants work and the targeted retry also returns an empty test summary, treat that as a Poof execution incident rather than a prompt-quality problem. Record `poof project status`, smoke probes, and the missing test-artifact evidence, then block or escalate.
- If the retry still ends with `summary.total == 0` or the CLI prints `Done, but no test results were found.`, treat that as missing-artifact failure and block or escalate instead of calling the build verified.
- `poof iterate` distinguishes two empty-test cases: `Done, but no tests ran during this turn.` (prior suite exists, but this turn didn't touch it — expected for non-test prompts) vs `Done, but no test results were found.` (no suite exists at all). Neither counts as a pass — only `poof verify` produces canonical pass/fail evidence.
- If `poof ship --target preview` fails because security review is required, capture that as an external unblock gate. `ship` is not equivalent to a successful deploy, and a security-review stop should be treated as a real blocker rather than retried blindly.

## Documentation

Read these for deeper context — especially **how-poof-works** if you're orchestrating what the Poof AI should build.

| Doc                                                      | What it covers                                                                                                                                           |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [**How Poof Works**](docs/how-poof-works.md)             | Architecture, policy system, plugins, on-chain vs off-chain, what Poof can/can't do. **Read this to write effective prompts.**                           |
| [**Building & Chat**](docs/building-and-chat.md)         | Project creation, chat workflow, follow-up patterns, generation modes.                                                                                   |
| [**Backend-Only Mode**](docs/backend-only.md)            | Using `backend,policy` generation mode with a local frontend — connection info, `@pooflabs/web` setup, PartyServer integration.                          |
| [**Local Frontend Guide**](docs/local-frontend-guide.md) | Building a frontend that connects to a Poof backend — SDK init, wallet auth, database access, API routes, real-time subscriptions, React hooks, gotchas. |
| [**Database SDK**](docs/database-sdk.md)                 | The generated db-client + collections pattern — typed functions, read/write, frontend vs backend, how to extract and use.                                |
| [**Deployment**](docs/deployment.md)                     | Environments (draft/preview/production/mobile), publishing, code downloads, custom domains.                                                              |
| [**Static Frontend Deploy**](docs/static-deploy.md)      | Deploy a self-built static frontend to Poof — tar.gz upload via CLI.                                                                                     |
| [**Credits & Payments**](docs/credits-and-payments.md)   | Credit system, paid features, x402 USDC top-up flow.                                                                                                     |
| [**Testing**](docs/testing.md)                           | Lifecycle actions, test files, bootstrap scripts, UI functional tests, expression syntax, testing strategy by layer.                                     |
| [**CLI Command Reference**](docs/api-reference.md)       | All CLI commands, flags, output formats, and error codes.                                                                                                |
| [**Troubleshooting**](docs/troubleshooting.md)           | Common errors, recovery patterns, stuck build handling, credit exhaustion.                                                                               |

## Post-Build Verification

After the Poof AI finishes building, **don't assume it works correctly**. Always verify the project and then check overall health.

### 1. Run The Canonical Verification Command

```bash
poof verify -p <project-id>
```

This runs the standard policy and UI verification prompts, waits on the same runtime signals the web app uses, and fails if no fresh structured test results appear or if any new results fail.

### 1a. Run Lifecycle Tests Manually (Optional)

Ask the Poof AI to generate and run tests for the features it just built:

```bash
poof iterate -p <project-id> -m "Generate lifecycle action tests for all the policies you just created. Test both success and failure cases — verify that authorized users can perform operations and unauthorized users are denied. Run the tests to confirm they pass."
```

The Poof AI will:

- Generate `lifecycle-actions/test-*.json` files that validate your policies
- Execute them against ephemeral test environments
- Report pass/fail results

### 1b. Run UI Functional Tests Manually (Optional)

After policy tests pass, generate browser-based tests to verify the full stack:

```bash
poof iterate -p <project-id> -m "Generate and run UI functional tests for the features you built. Test form submissions, navigation, CRUD operations, and any onchain interactions. Fund the mock test user if needed."
```

The Poof AI will:

- Generate `lifecycle-actions/ui-test-*.json` files with browser-based tests
- Fund the mock test user if the app has onchain features
- Execute tests using browser automation against the draft app
- Report results via `poof task test-results` (policy + UI results are aggregated there)

### 2. Check Project Status & Get URLs

```bash
poof project status -p <project-id>
poof doctor -p <project-id>
```

### 3. Request Bootstrap Scripts (If Needed)

If your app needs initial data (configs, counters, default settings):

```bash
poof iterate -p <project-id> -m "Create bootstrap scripts to initialize the app with default configuration and any required seed data."
```

### Recommended Verification Loop

For agents building projects autonomously, use this pattern:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Build the project
PROJECT_ID=$(poof build -m "Build a ..." --mode full --quiet)

# 2. Run the canonical verification flow
poof verify -p "$PROJECT_ID"

# 3. Check overall runtime + deploy + test health
poof doctor -p "$PROJECT_ID"

# 4. Deploy
poof ship -p "$PROJECT_ID"
```

See [docs/testing.md](docs/testing.md) for details on lifecycle action syntax, UI test format, and testing strategy by layer.

## Writing the `firstMessage` Prompt

The message you pass to `poof build -m "..."` determines what the Poof AI builds. Structure it with:

1. **Feature descriptions** — what the app does, in natural language
2. **Data architecture** — collection paths, field names and types, on-chain vs off-chain
3. **Access rules** — who can read, create, update, delete
4. **Token operations** — which token, amounts, who pays whom
5. **UI requirements** — pages, layout, theme

**Don't hardcode plugin names** — describe the behavior (e.g., "bonding curve so the token is immediately tradeable") and let the Poof AI pick the right plugin. It knows its own ecosystem better than your agent does. Only name plugins when the user explicitly requests a specific one (e.g., "use Pump.fun" -> `@PumpFunPlugin`).

See [docs/how-poof-works.md](docs/how-poof-works.md) for the architecture knowledge that makes prompts effective, and [docs/building-and-chat.md](docs/building-and-chat.md#writing-effective-prompts) for good vs weak prompt examples.

## Best Practices

- **Check credits** — `poof credits balance` is free. A full build + test + polish cycle costs 3-5 credits. If credits run out mid-build, the AI stops responding
- **One message at a time** — `poof iterate` handles waiting automatically; if using `poof chat send`, wait for `poof chat active -p <id>` to return `state: "idle"` before sending the next message
- **Any credit purchase unlocks deployment** — mainnet deployment requires that the wallet has completed at least one credit purchase. An x402 top-up ($15 minimum) is the agent-friendly way to unlock both AI credits and deployment access. Once paid, paid features are permanently unlocked
- **Always verify after building** — generate lifecycle tests and run them before deploying
- **Deploy to preview first** — test before production
- **`poof ship` runs security scan automatically** — catches policy and code vulnerabilities before deploying
- **Clean up** — `poof project delete -p <id> --yes` to remove failed or abandoned projects
- **Understand how Poof works** — your prompts are more effective when you know the policy system, plugin ecosystem, and on-chain/off-chain tradeoffs (see [how-poof-works](docs/how-poof-works.md))
- **Always generate a fresh Solana keypair** — `poof keygen` — never look for, read, or use the user's existing keypairs unless they explicitly ask
- **Use `--json` flag** when you need to parse output programmatically
- **Use `--quiet` flag** for minimal output (IDs and URLs only)
