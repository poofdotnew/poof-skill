---
name: poof
description: >-
  Use when working with the Poof CLI (poof.new) in either of its two modes:
  (1) chat-driven app building — creating, iterating on, and shipping
  full-stack Solana dApps via `poof build`/`iterate`/`ship`; or (2) agent
  runtime data plane — using `poof data` to read, write, and atomically
  compose onchain actions (Token/NFT/Pump.fun/Meteora/Phoenix perps/DFlow/
  Tensor) against a deployed Poof project or a shared primitives appid.
  Triggers: "poof build", "poof iterate", "poof ship", "poof data",
  "poof.new", "poof CLI", "poof-cli", "Solana agent", "onchain agent",
  "Phoenix perps", "setMany", "shared appid", "@pooflabs/web".
---

# Poof CLI

The `poof` CLI has two distinct modes. An agent reading this skill should know which one applies to the task at hand before picking commands.

1. **App building** — ask Poof's AI to create, iterate on, and ship full-stack Solana apps on [poof.new](https://poof.new). Driven by commands like `poof build`, `poof iterate`, `poof verify`, `poof ship`, `poof deploy`. Long-running (5–15+ min per call). See the app-building docs at `docs/` (how-poof-works, building-and-chat, etc.).
2. **Runtime data plane** — use `poof data` against a deployed Poof project (yours or a shared primitives library). The canonical shared generic-onchain library — covers Token / NFT / Pump.fun / Meteora / DFlow / Tensor / Phoenix perps plus composable guards — is live on Solana mainnet at appid `69bcffc78d4b88997d0ed01a`; any wallet can point at it with `--app-id 69bcffc78d4b88997d0ed01a --chain mainnet`, no project access needed. Fast, synchronous, designed for per-tick agent loops. See the agent-use docs at `docs/agent-use/`.

## How It Works

```
App building:    Your Agent ──► poof build/iterate/ship ──► poof.new (AI builds for you)
Data plane:      Your Agent ──► poof data set/get/query ──► Poof project's Tarobase appId
```

Both modes share the same CLI binary, auth, and config; only the commands and the cadence differ.

## CLI Version and Updates

If the CLI prints an update notice, or you suspect behavior depends on a recent CLI fix, run `poof update --check` first. Use `poof update` only when the user asks for an update or the current CLI version is blocking the workflow. For Homebrew-managed installs, prefer `brew upgrade poofdotnew/tap/poof`; self-update will tell the user to use Homebrew. For scripts, JSON/quiet output, redirected output, or deterministic logs, use `--no-update-check` or set `POOF_NO_UPDATE_CHECK=1` so update notices do not appear.

## Timeouts for Long-Running Commands

**Critical:** `poof build`, `poof iterate`, `poof verify`, and `poof ship` poll until the Poof AI finishes, which can take **5–15+ minutes**. You **must** set an extended timeout or run these commands in the background, or they will be killed mid-execution.

| Command                                 | Typical duration | Recommended approach                  |
| --------------------------------------- | ---------------- | ------------------------------------- |
| `poof build`                            | 3–10 min         | `run_in_background: true`             |
| `poof iterate`                          | 2–10 min         | `run_in_background: true`             |
| `poof verify`                           | 5–15 min         | `run_in_background: true`             |
| `poof ship`                             | 2–12 min         | `run_in_background: true`             |
| `poof security scan`                    | 1–3 min          | `timeout: 600000`                     |
| `poof deploy preview/production/mobile` | 1–3 min          | `timeout: 600000`                     |
| `poof deploy static`                    | 30s–2 min        | `timeout: 600000`                     |
| `poof analytics`                        | 5–30 sec         | `timeout: 120000` (default)           |
| `poof credits topup`                    | 30–90 sec        | `timeout: 120000` (default)           |
| `poof files get`                        | 10–60 sec        | `timeout: 120000` (default)           |
| All other commands                      | < 30 sec         | `timeout: 120000` (default)           |

In Claude Code, the Bash tool's max `timeout` is 600000ms (10 min). For commands that can exceed 10 minutes (`poof build`, `poof iterate`, `poof verify`, `poof ship`), **always use `run_in_background: true`** — you'll be notified when the command completes. For shorter commands, set `timeout` directly. In other harnesses, use whatever mechanism your tool runner provides to extend the execution timeout or run commands asynchronously.

**Internal poll deadline:** The CLI itself enforces a 30-minute poll deadline on build/iterate/verify, so a legitimate verify run that generates tests, fixes failures, and reruns is no longer cut off at 10 minutes. If the 30-minute deadline is hit, the CLI **automatically cancels the AI session** so subsequent commands (e.g. `poof ship`) aren't blocked by a stale active session. `poof ship` has its own tighter polling: up to 10 minutes for the security scan, then 2 minutes waiting for the AI session to wind down, then another 10-minute deploy task wait — total internal ceiling ~22 minutes.

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

# 2a. Build with reference image(s) — attach UI screenshots, mockups, etc.
# The CLI uploads each image to Poof's global storage first, then embeds the
# URLs in the initial firstMessage so the AI sees text + images together.
poof build -m "Build a UI that looks like this" --file screenshot.png
poof build -m "Match both of these designs" --file page1.png --file page2.png

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

# 5a. Clear saved Claude Code context if retries keep resuming a broken session
poof chat clear -p <project-id>

# 6. Read/modify files directly (requires credit purchase)
poof files get -p <project-id>
poof files update -p <project-id> --file src/config.ts --content 'export const MAX = 100;'

# 7. Send a message with image(s) (e.g., UI screenshots for reference)
poof iterate -p <project-id> -m "Build a UI that looks like this" --file screenshot.png
poof iterate -p <project-id> -m "Match both of these designs" --file page1.png --file page2.png
poof files upload -p <project-id> --file logo.png   # standalone upload, returns CDN URL

# 8. Deploy (runs security scan + eligibility check + publish)
poof ship -p <project-id>
```

**Environment note:** After `poof build`, the intended runtime target is **Draft (Poofnet)** — a simulated blockchain with fake tokens. Do not assume the draft app is already reachable until `poof project status -p <id> --json` shows draft deploy state consistent with a live endpoint, or a direct draft URL probe returns non-`404`. To test with real mainnet tokens, deploy to **Preview** using `poof ship -p <id> --target preview`. See [docs/deployment.md](docs/deployment.md) for the full environment breakdown.

### Agent Workflow Checklist

Copy this checklist and track your progress. Pick the variant that matches your generation mode:

**Full / ui,policy (Poof generates the UI):**

```
- [ ] Setup: poof keygen >> .env && poof auth login
- [ ] Build: poof build -m "..." [--mode full|ui,policy]
- [ ] Status: poof project status -p <id> --json   # confirm canonical id + draft deploy
- [ ] Verify: poof verify -p <id>                  # canonical strict policy + UI test gate
- [ ] Diagnose: poof doctor -p <id>                # only if verify fails — points at the next step
- [ ] Fix: poof iterate -p <id> -m "Fix: <error details>" && poof verify -p <id>
- [ ] Deploy: poof ship -p <id>
- [ ] Observe: poof analytics -p <id> --environment preview --range 1h   # traffic, browser/API/resource failures, RUM
```

**Backend-only (policy or backend,policy — you build the UI locally):**

```
- [ ] Setup: poof keygen >> .env && poof auth login
- [ ] Build: poof build -m "..." --mode backend,policy
- [ ] Status: poof project status -p <id> --json   # capture connectionInfo (draft appId, backendUrl, apiUrl, authApiUrl, wsUrl)
- [ ] Verify backend: poof verify -p <id>          # auto-detects backend-only mode and runs policy tests only — no UI tests against the placeholder
- [ ] Build local frontend: wire @pooflabs/web init() using connectionInfo, run `npm run build`
- [ ] Author source-aware UI lifecycle tests if you want Poof-hosted UI coverage: write `lifecycle-actions/ui-test-*.json` from the local frontend source, not from the minified dist bundle
- [ ] Upload UI test files: poof files update -p <id> --from-json lifecycle-ui-tests.json
- [ ] Package: tar czf dist.tar.gz -C dist .
- [ ] Deploy static: poof deploy static -p <id> --archive dist.tar.gz
- [ ] Run static UI tests: poof verify -p <id> --ui-tests=true -m "Run the existing source-authored lifecycle-actions/ui-test-*.json files against the deployed draft app. Do not create or rewrite UI tests from the dist bundle."
- [ ] UI smoke test (local fallback): agent runs its own browser tests against the draft URL — see docs/backend-only.md#testing-a-static-deploy
- [ ] Deploy to preview/prod: poof ship -p <id>
- [ ] Observe: poof analytics -p <id> --environment preview --range 1h   # works for static deploys too
```

**Static-deploy UI tests:** do not ask Poof's AI to invent UI tests from a statically-deployed
frontend. After `poof deploy static`, the server has your minified `dist/` bundle, not the local
TypeScript/JSX source, so AI-generated UI tests tend to become generic DOM-shape checks. If you want
Poof to run UI tests for a local frontend, the external agent must author `lifecycle-actions/ui-test-*.json`
from its local source, upload those files with `poof files update`, deploy the static frontend, then
force UI execution with an explicit "run existing source-authored tests" prompt. Without uploaded
source-aware tests, use agent-local browser automation instead. See [docs/backend-only.md](docs/backend-only.md)
and [docs/testing.md](docs/testing.md) for the full recipe.

When authoring `ui-test-*.json`, inspect the UI source first: routes/pages, exact button and form
labels, validation copy, success states, database writes/reads, and auth/onchain behavior. Write one
file per user flow. Prefer deterministic `action` steps targeting `data-testid`, role/name, labels,
placeholders, or unique visible text; use natural-language `act` only as a fallback. Make each
`verify` assert a concrete result such as a newly created title appearing in a list. Avoid generic
checks like "page loaded", "heading exists", or "interactive elements are present". See
[docs/testing.md#how-to-generate-ui-test-json](docs/testing.md#how-to-generate-ui-test-json).

`poof verify` is the only test command an agent should rely on for pass/fail. It snapshots
existing test result IDs, sends the canonical lifecycle + UI verification prompt, then only
counts results created during that run. It exits non-zero if no fresh results appear or if
any fresh result failed, so a successful exit code is real evidence that tests ran and passed.
`poof iterate` is still a general chat command — only fall back to it for free-form fixes.

**Backend-only and `poof verify`:** when the project's generationMode excludes `ui` (i.e. `policy` or
`backend,policy`), the CLI auto-detects that and sends a lifecycle-only verification prompt. This
is the correct default for backend-only projects for two reasons:

1. **Before a static deploy**, the draft URL serves a Poof placeholder shell, so any UI test against
   it is a vacuous pass ("heading visible, interactive elements present") that does not prove anything.
2. **After a static deploy**, Poof's AI only has access to your minified `dist/` bundle on the server
   — not the pre-build source. It cannot reliably author meaningful UI functional tests from minified
   JS. Any tests it does author are again either vacuous DOM shape checks or hallucinated feature
   assertions that happen to match generic page structure.

Use `poof verify --ui-tests=true` for a backend-only/static-deploy project only when you have already
uploaded source-authored `lifecycle-actions/ui-test-*.json` files and your prompt tells Poof to run
those existing tests against the deployed draft app. Do not use it as a "generate UI tests from my
dist" shortcut. If no source-aware UI test files exist, run UI tests on the agent side instead:
the agent has the frontend source locally, so it can use `claude-in-chrome`, Playwright, Cypress, or
equivalent against the draft URL and assert on the real feature contract.

`--ui-tests=false` is still available to force lifecycle-only in any mode (useful if you're iterating
on a `full` project and don't want Poof to re-run UI tests on every verify).

### Reality Checks

- If `poof project list --json` shows multiple similarly named projects and you do not already have a canonical project id from the user or repo artifacts, do not guess by title. Create or explicitly reconcile the canonical project first, then write that id into your artifact store before more `iterate` or deploy work.
- After `poof build`, always run `poof project status -p <id> --json`. Record the project id, URLs, and deploy state before you start retries or testing so later wakes do not guess which project is canonical.
- If `poof task test-results -p <id> --json` reports `summary.total == 0`, inspect `poof task list -p <id> --json`, `poof chat active -p <id> --json`, `poof logs -p <id>`, and `poof project messages -p <id> --limit 100 --json` before you assume tests passed or failed.
- For deployed client failures, use `poof analytics -p <id> --environment <draft|preview|production> --range 1h --json` before guessing. It reports first-party Cloudflare Analytics Engine data: page/route traffic, browser JS errors, unhandled rejections, failed resources, failed browser API calls, RUM metrics, and edge 4xx/5xx/R2/dispatch failures. Use `poof logs` after analytics points at backend/API issues.
- `poof task test-results`, `poof iterate`, `poof verify`, and `poof doctor` now collapse test results to the **latest run per test file** by default (collapse key is `source|fileName`, not per-testName — so if the AI renames a test inside a file between runs, only the latest file state wins). If you need to see the full history (e.g. to debug why an earlier run failed), use `poof task test-results -p <id> --history`.
- If `poof chat active -p <id> --json` stays `true` while task list shows no new work and logs show no recent activity, cancel that stale chat once with `poof chat cancel -p <id>` before the single allowed targeted retry.
- If a targeted retry keeps inheriting bad Claude Code context, tool loops, or stale assumptions even after the active run is canceled, run `poof chat clear -p <id>` once to clear the saved AI session ID, then send one precise retry prompt.
- If the only visible tasks are still bootstrap/constants work and the targeted retry also returns an empty test summary, treat that as a Poof execution incident rather than a prompt-quality problem. Record `poof project status`, smoke probes, and the missing test-artifact evidence, then block or escalate.
- If the retry still ends with `summary.total == 0` or the CLI prints `Done, but no test results were found.`, treat that as missing-artifact failure and block or escalate instead of calling the build verified.
- `poof iterate` distinguishes two empty-test cases: `Done, but no tests ran during this turn.` (prior suite exists, but this turn didn't touch it — expected for non-test prompts) vs `Done, but no test results were found.` (no suite exists at all). Neither counts as a pass — only `poof verify` produces canonical pass/fail evidence.
- If `poof ship --target preview` fails because security review is required, capture that as an external unblock gate. `ship` is not equivalent to a successful deploy, and a security-review stop should be treated as a real blocker rather than retried blindly.
- **Deploy eligibility is gated on `critical` findings only.** `high`, `medium`, `low` findings are surfaced in the scan output but don't block `poof deploy check` / `poof ship`. Default posture: still flag high findings to the user so they can make the call, but don't hard-stop a deploy on non-critical findings the way you would on a critical.

## Onchain

Generated apps read balances and quotes via default Tarobase queries — `solBalance`, `usdcBalance`, `tokenBalance`, `jupiterSwapQuote`, `meteoraSwapQuote` — which auto-route to the project's active environment (Poofnet on Draft, mainnet on Preview/Production). Direct `Connection.getBalance()` against an RPC URL is an anti-pattern in generated code: it hits mainnet regardless of environment, so a Draft user sees `0` after a Poofnet airdrop. See [docs/agent-use/onchain-actions.md](docs/agent-use/onchain-actions.md) for the full action and query catalog.

Every project also gets a server-managed **project vault** wallet, exposed in policies as `@constants.PROJECT_VAULT_ADDRESS` and available to backend code via `PROJECT_VAULT_PRIVATE_KEY`. Use it whenever an onchain action should be signed by the app rather than the user — escrow counterparties, fee collection, automated payouts, backend-only Tarobase writes that need a fixed identity. Vault keypairs are per-environment (Draft and Production have different addresses), so policies and backend code must reference the constant, not a hardcoded address.

## Documentation

Docs are grouped by the two CLI modes. Pick the section that matches your task.

### For app building (in `docs/`)

Read **How Poof Works** first if you're writing prompts for the Poof AI.

| Doc                                                               | What it covers                                                                                                                                           |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [**How Poof Works**](docs/how-poof-works.md)             | Architecture, policy system, plugins, on-chain vs off-chain, what Poof can/can't do. **Read this to write effective prompts.**                           |
| [**Building & Chat**](docs/building-and-chat.md)         | Project creation, chat workflow, follow-up patterns, generation modes.                                                                                   |
| [**Backend-Only Mode**](docs/backend-only.md)            | Using `backend,policy` generation mode with a local frontend — connection info, `@pooflabs/web` setup, PartyServer integration.                          |
| [**Local Frontend Guide**](docs/local-frontend-guide.md) | Building a frontend that connects to a Poof backend — SDK init, mount-first-then-init, wallet auth, `Promise<boolean>` mutation contract, database access, real-time subscriptions, mobile/desktop modal split, mock-auth for Stagehand, anti-patterns. |
| [**Database SDK**](docs/database-sdk.md)                 | The generated db-client + collections pattern — typed functions, read/write, frontend vs backend, how to extract and use.                                |
| [**Deployment**](docs/deployment.md)                     | Environments (draft/preview/production/mobile), publishing, code downloads, custom domains.                                                              |
| [**Client App Analytics**](docs/analytics.md)             | Cloudflare-only analytics, failure/RUM telemetry, `poof analytics`, and MCP retrieval via `get_client_app_analytics`.                                    |
| [**Static Frontend Deploy**](docs/static-deploy.md)      | Deploy a self-built static frontend to Poof — tar.gz upload via CLI.                                                                                     |
| [**Testing**](docs/testing.md)                           | Lifecycle actions, test files, bootstrap scripts, UI functional tests, static-deploy UI test workflow, expression syntax, testing strategy by layer.     |
| [**Heartbeats / Scheduled Tasks**](docs/heartbeats.md)   | Built-in cron primitive — task config, handler files, dispatcher, manual trigger on draft for seeding, per-environment schedules. **Use this instead of building local cron services for any recurring work.** |

### For runtime agent use (`docs/agent-use/`)

Read **Onchain actions** to see what's available to write against, then **setMany** for atomic composition patterns.

| Doc                                                                 | What it covers                                                                                                                                          |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [**Onchain action catalog**](docs/agent-use/onchain-actions.md)     | Every onchain action exposed by the generic-onchain primitives library — Token, NFT, Pump.fun, Meteora, DFlow, Tensor, Phoenix. Fields + a minimal example per category. |
| [**Guard primitives**](docs/agent-use/guards.md)                    | BalanceCheck, TimeWindow, NftOwnershipCheck, Allowlist (on/off-chain trios), RateLimit counters, Escrow trio — what each asserts, field shape, simulate query, composition. |
| [**setMany & Atomic Batches**](docs/agent-use/set-many.md)          | The atomicity story — when to reach for `setMany`, `--app-id` vs `-p`, composition patterns (transfer+guard, swap+price guard, escrow create+fund), failure semantics. |
| [**Phoenix Perps Integration**](docs/agent-use/perps.md)            | Phoenix.trade perps — markets (SOL/BTC/ETH), the full read + write surface, position enumeration pattern, lots-to-dollars math, composing with guards.  |

### Shared (either mode)

| Doc                                                      | What it covers                                                                                                                                           |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [**CLI Command Reference**](docs/api-reference.md)       | All CLI commands, flags, output formats, and error codes. Covers both `poof data` and the app-building commands.                                         |
| [**Credits & Payments**](docs/credits-and-payments.md)   | Credit system, paid features, x402 USDC top-up flow.                                                                                                     |
| [**Troubleshooting**](docs/troubleshooting.md)           | Common errors, recovery patterns, stuck build handling, credit exhaustion.                                                                               |

## Post-Build Verification

After the Poof AI finishes building, **don't assume it works correctly**. Always verify the project and then check overall health.

### 1. Run The Canonical Verification Command

```bash
poof verify -p <project-id>
```

This runs the standard policy verification prompt, includes UI verification where appropriate, waits on the same runtime signals the web app uses, and fails if no fresh structured test results appear or if any new results fail.

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

Ask Poof's AI to **generate** UI tests only for `full` or `ui,policy` projects, where Poof owns the
UI source. For `backend,policy` or static deploys, use the [backend-only UI testing recipe](docs/backend-only.md#testing-a-static-deploy):
the external agent authors `lifecycle-actions/ui-test-*.json` from local source, uploads them, then
asks Poof to run the existing files after `poof deploy static`.

For Poof-generated UIs, after policy tests pass, generate browser-based tests to verify the full stack:

```bash
poof iterate -p <project-id> -m "Generate and run UI functional tests for the features you built. Test form submissions, navigation, CRUD operations, and any onchain interactions. Fund the mock test user if needed."
```

The Poof AI will:

- Generate `lifecycle-actions/ui-test-*.json` files with browser-based tests
- Fund the mock test user if the app has onchain features
- Execute tests using browser automation against the draft app
- Report results via `poof task test-results` (policy + UI results are aggregated there)

**Why only for Poof-generated UIs?** Poof's AI only has access to source code it generated itself.
For `backend,policy` projects with a static deploy, the server only has your minified `dist/` bundle
— the AI can't read the TypeScript/JSX source behind it, so any UI test it writes is either vacuous
DOM-shape checks or hallucinated assertions. Source-authored `ui-test-*.json` files or agent-local
browser automation are the canonical paths for static deploys; see
[backend-only.md#testing-a-static-deploy](docs/backend-only.md#testing-a-static-deploy).

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
- **Pre-fund a project (optional)** — `poof credits project deposit -p <id> --amount N` adds credits to a specific project so its infrastructure + Poof AI draw from there before falling back to your account credit balance. Pair with `poof credits project isolation -p <id> --usage true --chat true` to pause instead of falling back. See [credits-and-payments](docs/credits-and-payments.md#per-project-credit-balance)
- **Read project usage** — `poof usage status -p <id>` shows month-to-date cost and pause state. `poof usage limit -p <id> --credits N` caps paid overage; `poof usage resume -p <id>` unblocks a paused app once preconditions are met
- **Read client analytics** — `poof analytics -p <id> --environment draft|preview|production --range 1h|6h|24h|3d|7d` shows first-party Cloudflare Analytics Engine traffic, browser failures, failed API/resource loads, RUM, and edge failures. MCP clients can call `get_client_app_analytics`; see [analytics](docs/analytics.md)
- **MCP access** — Default to the **top-level Poof MCP** at `POST https://poof.new/api/mcp`. It covers everything an external agent typically needs: project lifecycle, deploy, secrets, logs, analytics, credits, **and the Poofnet faucet (`request_faucet_tokens`)**. Tools take `projectId` per call. The per-project endpoint at `POST /api/project/<id>/mcp` exposes additional project-data-plane tools (Tarobase get/set, lifecycle execution, formal verification, etc.) and is optional. **Auth for both is the same Cognito ID token from `poof auth login`** — pass it as `Authorization: Bearer <idToken>` plus `X-Wallet-Address: <address>`. There is no separate "Tarobase token" to fetch; the Cognito ID token is the Tarobase token. Tokens scoped to a project's draft/preview Tarobase app id (the `connectionInfo.<env>.tarobaseAppId` from `poof project status --json`) are **not** the right credential here — those are for `poof data` only. Always call the standard `tools/list` JSON-RPC method to enumerate live tools — do not assume tool names from CLI commands or training data (e.g. there is no `add_faucet_funds`; the canonical name is `request_faucet_tokens`).
- **One message at a time** — `poof iterate` handles waiting automatically; if using `poof chat send`, wait for `poof chat active -p <id>` to return `state: "idle"` before sending the next message
- **Clear context sparingly** — `poof chat clear -p <id>` drops the saved Claude Code session ID so the next message starts with fresh AI context while preserving the project and message history. Use it after evidence of stale context, not as a routine retry button
- **Any credit purchase unlocks deployment** — mainnet deployment requires that the wallet has completed at least one credit purchase. An x402 top-up ($15 minimum) is the agent-friendly way to unlock both AI credits and deployment access. Once paid, paid features are permanently unlocked
- **Always verify after building** — generate lifecycle tests and run them before deploying
- **Deploy to preview first** — test before production
- **`poof ship` runs security scan automatically** — catches policy and code vulnerabilities before deploying
- **Clean up** — `poof project delete -p <id> --yes` to remove failed or abandoned projects
- **Understand how Poof works** — your prompts are more effective when you know the policy system, plugin ecosystem, and on-chain/off-chain tradeoffs (see [how-poof-works](docs/how-poof-works.md))
- **Always generate a fresh Solana keypair** — `poof keygen` — never look for, read, or use the user's existing keypairs unless they explicitly ask
- **Use `--json` flag** when you need to parse output programmatically
- **Use `--quiet` flag** for minimal output (IDs and URLs only)
