# Building & Chat

The `poof iterate` command is the primary way your agent interacts with Poof projects. It sends a chat message to the Poof AI, waits for it to finish, and shows test results.

## Contents

- [Project Lifecycle](#project-lifecycle)
- [Writing Effective Prompts](#writing-effective-prompts)
- [Updating Project Settings](#updating-project-settings)
- [End-to-End Example](#end-to-end-example)

## Project Lifecycle

### 1. Create a Project

```bash
# Build with default mode (full — UI, backend, policies, deployment)
poof build -m "Build a token-gated content platform with USDC payments"

# Or specify a generation mode
poof build -m "Create a staking vault policy with 30-day lock" --mode policy

# Save project ID for later use
PROJECT=$(poof build -m "Build a token-gated content platform" --json | jq -r '.projectId')
```

The `-m` message is the initial prompt that tells the Poof AI what to build. Be specific about features, data models, and blockchain operations. The `poof build` command creates the project and **waits automatically** until the AI finishes.

### Generation Mode

Control what the AI generates with `--mode`:

| Mode             | What gets generated                                                        |
| ---------------- | -------------------------------------------------------------------------- |
| `full` (default) | Everything — UI, backend, policies, lifecycle actions                      |
| `policy`         | Database policies only (schema, rules, hooks, queries) + lifecycle actions |
| `ui,policy`      | Frontend UI + policies (no backend API routes)                             |
| `backend,policy` | Backend API routes + policies (no frontend UI)                             |

> **Note:** Bare values `ui` and `backend` are also accepted and automatically include policy (e.g., `--mode backend` is equivalent to `--mode backend,policy`).

#### Choosing the right mode

- **`full`** — Best when you want Poof to handle everything end-to-end. Use for standalone web apps where Poof owns the entire stack.
- **`policy`** — Use when building your own frontend or backend locally (React Native, Next.js, custom server). After creation, download the generated typed SDK via `poof deploy download`. Almost every project needs policies.
- **`ui,policy`** — Use when you want Poof to generate frontend and database policies, but handle backend yourself.
- **`backend,policy`** — Use when building your own frontend but want Poof to generate backend API routes and policies. After creation, get connection info via `poof project status -p <id> --json`. See [backend-only.md](backend-only.md).

#### Auth requirement for policies

If your app uses policies with access rules beyond `true` (rules that check caller identity), your frontend **must** integrate Tarobase login via the `@pooflabs/web` SDK so writes are wallet-signed. Without this, writes will be rejected.

Exception: policies where all write rules are `true` (fully public access) don't require authentication.

### 2. Wait for the AI to Finish

`poof build` and `poof iterate` wait automatically with built-in timeout and exponential backoff. This polling can take **5–10+ minutes** — see the timeout table in [SKILL.md](../SKILL.md) for recommended shell timeout values per command. In Claude Code, use `run_in_background: true` for `poof build`, `poof iterate`, and `poof ship` to avoid hitting the 10-minute Bash tool limit.

For long prompts with quotes, JSON, or multiple paragraphs, prefer `--stdin` or a quoted temp file instead of one huge inline `-m "..."` string. If you need the exit code afterward in zsh, use `rc=$?`, not `status=$?`.

If using the lower-level `poof project create` or `poof chat send` commands, check status manually:

```bash
poof chat active -p <project-id>
# Returns: { active: boolean, state: "running" | "queued" | "idle", status: "ok" | "error" }
```

### 3. Check Project Status & Get URLs

```bash
poof project status -p <project-id>

# For programmatic access:
poof project status -p <project-id> --json | jq '.urls'
```

The draft URL runs on Poofnet (simulated blockchain, free). Use `publishState.draft.deployed` from `poof project status --json` as the actual deploy/readiness signal; a draft URL can exist before the draft is serving traffic. See [deployment.md](deployment.md) for details.

### Execution Control

If the AI gets stuck or goes in the wrong direction:

```bash
# Option 1: Steer mid-execution (AI incorporates your message without stopping)
poof chat steer -p <project-id> -m "Focus on the backend policies, skip the UI for now"

# Option 2: Cancel and start fresh
poof chat cancel -p <project-id>
# Then send a new chat message
poof iterate -p <project-id> -m "Start over with just the voting logic"
```

### Direct File Editing

For quick fixes without going through chat:

```bash
# Read current files (requires a credit purchase)
poof files get -p <project-id>

# Modify specific files
poof files update -p <project-id> --file src/config.ts --content 'export const API_URL = "https://api.example.com";'

# Update multiple files from a JSON map
poof files update -p <project-id> --from-json files.json
```

### 4. Iterate via Chat

Send follow-up messages to refine the project:

```bash
poof iterate -p <project-id> -m "Add a leaderboard page showing top contributors by USDC spent"

cat <<'EOF' | poof iterate -p <project-id> --stdin
Add a gated chat flow with explicit eligible, grace, and revoked states.
Generate and run lifecycle plus UI tests after the implementation step.
EOF
```

`poof iterate` sends the message, waits for the AI to finish, and shows test results.

> **One message at a time.** `poof iterate` handles waiting automatically. If using the lower-level `poof chat send`, always wait for `poof chat active` to return `state: "idle"` before sending the next message. Sending while AI is active queues the message (FIFO), but the AI won't have your evaluation context.
>
> If `poof chat active -p <id> --json` stays `true` but `poof task list -p <id> --json` shows no new task ids and `poof logs -p <id>` shows no recent activity, treat that as stale active-chat state rather than healthy progress. Capture the evidence, run `poof chat cancel -p <id>`, then send at most one targeted retry message.

### 5. Get Conversation History

```bash
poof project messages -p <project-id>

# JSON output for parsing
poof project messages -p <project-id> --json
```

## Writing Effective Prompts

Since chat is your main interface, prompt quality matters. Knowing [how Poof works](how-poof-works.md) helps.

### Good Prompts (Specific, Poof-Aware)

- "Add a staking vault where users lock SOL for 30 days. Use @AccountPlugin for the escrow PDA. On-chain with passthrough for the unstake operation."
- "Create a tipping system using passthrough — no stored records, just direct USDC transfers from tipper to creator."
- "Add user profiles (off-chain) with name, bio, and avatar. Owner-only write access. Public reads."
- "Add verifiable coin flip gambling with VRF. 2x payout on win, SOL stakes."

### Weak Prompts (Vague, May Miss Optimizations)

- "Add staking" — unclear what token, duration, mechanism
- "Add a database" — Poof IS the database; be specific about what data
- "Make it work on Ethereum" — impossible; Solana only

### Tips

1. **Specify on-chain vs off-chain** when it matters for cost/privacy
2. **Name the plugins** if you know the right one (e.g., "@DeFiPlugin for Meteora bonding curve")
3. **Describe access patterns** — who can read, who can write
4. **Be explicit about token operations** — which token, amounts, who pays whom
5. **Mention passthrough** for operations that don't need persistent storage

## Updating Project Settings

```bash
poof project update -p <project-id> --title "My Token Platform" --description "A token-gated content platform"
```

## End-to-End Example

Complete bash workflow showing build, test, verify, fix, and deploy:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Setup (one-time)
poof keygen >> .env
poof auth login

# 2. Check credits
TOTAL=$(poof credits balance --json | jq '.credits.total')
if [ "$TOTAL" -lt 5 ]; then
  echo "Need 5+ credits, have $TOTAL"
  exit 1
fi

# 3. Build project
PROJECT_ID=$(poof build -m "Build a token-gated content platform where creators post articles and readers pay 0.1 USDC per article. User profiles (off-chain), articles (off-chain), passthrough payment (on-chain)." --json | jq -r '.projectId')
echo "Project: $PROJECT_ID"

# 4. Generate and run lifecycle tests
poof iterate -p "$PROJECT_ID" -m "Generate and run lifecycle action tests for all policies. Cover both allowed and denied operations."

# 5. Check test results — verify tests ran AND passed
TOTAL=$(poof task test-results -p "$PROJECT_ID" --json | jq '.summary.total')
FAILED=$(poof task test-results -p "$PROJECT_ID" --json | jq '.summary.failed')
ERRORS_COUNT=$(poof task test-results -p "$PROJECT_ID" --json | jq '.summary.errors')

if [ "$TOTAL" -eq 0 ]; then
  echo "Warning: no tests were generated. Retrying..."
  poof iterate -p "$PROJECT_ID" -m "No test results found. Please generate and run lifecycle action tests for all policies."
  TOTAL=$(poof task test-results -p "$PROJECT_ID" --json | jq '.summary.total')
  FAILED=$(poof task test-results -p "$PROJECT_ID" --json | jq '.summary.failed')
  ERRORS_COUNT=$(poof task test-results -p "$PROJECT_ID" --json | jq '.summary.errors')
fi

if [ "$FAILED" -gt 0 ] || [ "$ERRORS_COUNT" -gt 0 ]; then
  ERRORS=$(poof task test-results -p "$PROJECT_ID" --json | jq -r '[.results[] | select(.status=="failed" or .status=="error") | "- \(.fileName): \(.lastError // .status)"] | join("\n")')
  echo "Tests failed — sending fix request"
  poof iterate -p "$PROJECT_ID" -m "The following tests failed:

$ERRORS

Please fix the issues and re-run the tests."
fi

# 6. Generate and run UI functional tests
poof iterate -p "$PROJECT_ID" -m "Generate and run UI functional tests. Test form submissions, navigation, CRUD operations, and any onchain interactions. Fund the mock test user if needed."

# 7. Get project URLs
poof project status -p "$PROJECT_ID"

# 8. Deploy (runs security scan + eligibility check + publish)
poof ship -p "$PROJECT_ID"

echo "Done! Project deployed."
```
