# Troubleshooting

Common errors, causes, and CLI recovery patterns.

## Contents
- [Authentication Errors](#authentication-errors)
- [Build & Chat Errors](#build--chat-errors)
- [Deployment Errors](#deployment-errors)
- [x402 Payment Errors](#x402-payment-errors)
- [Credit Errors](#credit-errors)
- [Timeout Errors](#timeout-errors)
- [Agent Recovery Patterns](#agent-recovery-patterns)

## Authentication Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Not authenticated` | Invalid or expired session | Run `poof auth login` to re-authenticate |
| `Invalid projectId format` | Non-UUID project ID | Project IDs must be valid UUIDs — copy from `poof project list` output |
| Auth succeeds but operations fail | Environment mismatch | Check `POOF_ENV` — must be `production`, `staging`, or `local`. Omit for production (default) |

## Build & Chat Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Command killed / no output / timeout | Shell timeout too short for a polling command | `poof build`, `poof iterate`, and `poof ship` can take 5–10+ minutes. Set timeout to 600000ms (10 min) or use `run_in_background: true`. See the timeout table in SKILL.md |
| Build stuck for >10 minutes | AI is stuck or in a loop | Run `poof chat cancel -p <id>`, then start a new iteration with clearer instructions |
| Build finishes immediately | AI server may be unreachable | Check the exit status — a non-zero code means the server is down. Retry after a short delay, or run `poof project status -p <id>` for the latest task status |
| Build fails after starting | Session expired mid-build | Run `poof auth login` and retry |
| AI builds the wrong thing | Vague or ambiguous `-m` flag in `poof build` | Be specific about data models, access rules, token operations, and on-chain vs off-chain. See [how-poof-works.md](how-poof-works.md) |
| `Project not found` | Wrong project ID or not the owner | Verify project ID and that you're using the same keypair that created the project |

## Deployment Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Deploy blocked — `no_membership` | User has never purchased credits | The wallet needs at least one completed credit purchase. Run `poof credits topup` ($15 minimum). Once any purchase completes, deployment is permanently unlocked for that wallet |
| Deploy blocked — security review | Failed security scan | Fix flagged issues and retry. `poof ship` runs the security scan automatically |
| Preview deploy fails | Missing or expired auth | Run `poof auth login`, then retry `poof deploy` |
| Production deploy fails | Haven't passed eligibility check | Run `poof deploy check` first and resolve any blockers, or use `poof ship` which checks automatically |
| `poof credits topup` returns an error | Settlement or credit award failed | Run `poof credits balance` to see if the payment was partially processed. If charged on-chain but no credits, contact support with the txId |

## x402 Payment Errors

The CLI handles x402 payments internally. If `poof credits topup` fails, ensure your wallet has sufficient USDC on Solana mainnet and try again.

## Credit Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `You have run out of credits` | No credits remaining | Run `poof credits balance` — wait for daily reset or top up with `poof credits topup` |
| AI stops responding mid-build | Credits exhausted during execution | Run `poof credits balance`. If zero, top up and run `poof iterate -p <id>` to continue |
| Build silently does nothing | Insufficient credits to start | Always run `poof credits balance` before starting a workflow. A full build + test + polish cycle costs 3-5 credits |
| Can't deploy after buying credits | Payment may not have completed | Verify the payment actually completed — run `poof credits balance` and check for add-on records |

## Timeout Errors

The most common agent issue: the shell kills `poof build`, `poof iterate`, or `poof ship` before it finishes because the default timeout is too short.

**Symptoms:**
- Command produces no output or partial output
- Agent retries the same command and it fails again
- "Command timed out" or similar harness error
- Agent reports the command "hung" or "didn't respond"

**Root cause:** These commands poll until the Poof AI finishes, which can take 5-10+ minutes. Most tool harnesses default to 2-minute timeouts.

**Fix:** Set an extended timeout or run in background. See the timeout table in [SKILL.md](../SKILL.md) for recommended values per command.

| Command type | Recommended approach |
|-------------|---------------------|
| `poof build`, `poof iterate`, `poof ship` | `run_in_background: true` (preferred) or `timeout: 600000` |
| `poof security scan`, `poof deploy *` | `timeout: 600000` |
| All other commands | `timeout: 120000` (default is fine) |

**Example (Claude Code):**
```
# Background — best for build/iterate/ship (may exceed 10 min)
Bash tool: command="poof build -m '...'", run_in_background=true

# Extended timeout — for security scan, deploy
Bash tool: command="poof security scan -p <id>", timeout=600000
```

## Agent Recovery Patterns

### Stuck Build Recovery
```bash
poof chat cancel -p <id>
poof iterate -p <id> -m "The previous build got stuck. Review current state and continue."
```

### Credit Check Before Workflow
```bash
TOTAL=$(poof credits balance --json | jq '.credits.total')
if [ "$TOTAL" -lt 5 ]; then
  echo "Insufficient credits: ${TOTAL} remaining, need 5. Top up or wait for daily reset."
  exit 1
fi
```
