# Troubleshooting

Common errors, causes, and CLI recovery patterns.

## Contents
- [Authentication Errors](#authentication-errors)
- [Build & Chat Errors](#build--chat-errors)
- [Deployment Errors](#deployment-errors)
- [Client Analytics & Runtime Errors](#client-analytics--runtime-errors)
- [x402 Payment Errors](#x402-payment-errors)
- [Credit Errors](#credit-errors)
- [Timeout Errors](#timeout-errors)
- [CLI Version & Update Errors](#cli-version--update-errors)
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
| Command killed / no output / timeout | Shell timeout too short for a polling command | `poof build`, `poof iterate`, `poof verify`, and `poof ship` can take 5–15+ minutes. Use `run_in_background: true` in Claude Code or set `timeout: 600000`. See the timeout table in SKILL.md |
| `verify timed out or failed: timed out after 10m0s` | Pre-CLI-fix verify deadline | Run `poof update --check`, then update with `poof update` or `brew upgrade poofdotnew/tap/poof` for Homebrew installs. Current versions enforce a 30-minute internal poll deadline on build/iterate/verify |
| Build stuck for >15 minutes | AI is stuck or in a loop | Run `poof chat cancel -p <id>`, then start a new iteration with clearer instructions |
| `poof chat active` stays `true` but `task list`/`test-results` never change | Stale active-chat state after the AI stopped making progress | Run `poof task list -p <id> --json`, `poof logs -p <id>`, and `poof task test-results -p <id> --json`. If there is no new task and no recent activity, capture that evidence, run `poof chat cancel -p <id>`, then retry once with a targeted prompt |
| Targeted retries keep repeating stale assumptions or broken Claude Code context | Saved AI session ID is resuming an unhealthy context | Run `poof chat clear -p <id>` once, then send one precise retry prompt |
| Build finishes immediately | Rare edge case if AI starts and finishes very quickly, or server issue | Run `poof project status -p <id>` to verify the project has tasks and content. If the project looks empty, retry with `poof iterate -p <id> -m "..."` |
| Build fails after starting | Session expired mid-build | Run `poof auth login` and retry |
| AI builds the wrong thing | Vague or ambiguous `-m` flag in `poof build` | Be specific about data models, access rules, token operations, and on-chain vs off-chain. See [how-poof-works.md](how-poof-works.md) |
| `Project not found` | Wrong project ID or not the owner | Verify project ID and that you're using the same keypair that created the project |

## Deployment Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Deploy blocked — `no_membership` | User has never purchased credits | The wallet needs at least one completed credit purchase. Run `poof credits topup` ($15 minimum). Once any purchase completes, deployment is permanently unlocked for that wallet |
| Deploy blocked — security scan found issues | `poof ship` prints "Security scan finished" then fails eligibility | The scan completed but found vulnerabilities. Fix the flagged issues with `poof iterate`, then retry `poof ship`. The CLI distinguishes between scan completion (neutral) and scan passing (success) |
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
| App paused (`isBlocked: true`)                                    | Free tier or limit exhausted        | Read `blockedReason`: `no_overuse_limit` → `poof usage limit --credits N`; `threshold_reached` → raise the limit; `insufficient_credits` → `poof credits topup` or `project deposit`. Then `poof usage resume`. |
| `withdrawal already in progress`                                  | Concurrent withdraw                  | Wait — locks auto-clear after 5 minutes.                                                                                                                                                                          |
| `poof usage status` shows `summaryStale` / `blockedStatusStale`   | Upstream pipeline failed             | Don't act on the stale fields; retry shortly.                                                                                                                                                                     |

## Client Analytics & Runtime Errors

Use `poof analytics` when the symptom is client-facing: blank page, broken route, failed image/script,
browser JS error, API call failing from the UI, slow page load, or a reported deployed-app failure
that is not explained by task/test output.

```bash
poof analytics -p <id> --environment draft --range 1h --json
poof analytics -p <id> --environment preview --range 1h --json
poof analytics -p <id> --environment production --range 1h --json
```

| Symptom | What to check | Next step |
|---------|---------------|-----------|
| Blank or broken page | `summary.errors`, `errors[]` entries like `js_error`, `unhandled_rejection`, `resource_error`, `static_4xx`, `static_5xx` | Fix the UI/static asset problem, redeploy, then re-run analytics for the same range |
| UI action fails | `api_error`, `api_4xx`, `api_5xx`, `dispatch_error`, route bucket, status/failure class | Run `poof logs -p <id> --environment <env>` for backend details, then fix via `poof iterate` or source changes |
| Missing route in SPA | `spa_fallback`, `navigation_error`, `static_4xx`, top page path | Check routing and static deploy contents; verify direct URL smoke probes |
| App says disabled / 403 | First run `poof usage status -p <id>` and inspect `isBlocked` / `blockedReason` | Resolve usage/credit pause before debugging code |
| No analytics rows | New deploy, no traffic yet, wrong environment, or Analytics Engine dataset not initialized | Confirm URL/environment with `poof project status -p <id> --json`, generate a smoke visit, then retry shortly |

`poof logs` only shows backend/API Worker logs. It will not show browser runtime errors or failed
static resources; use `poof analytics` for those. See [analytics.md](analytics.md) for the complete
event catalog and privacy rules.

## Timeout Errors

The most common agent issue: the shell kills `poof build`, `poof iterate`, or `poof ship` before it finishes because the default timeout is too short.

**Symptoms:**
- Command produces no output or partial output
- Agent retries the same command and it fails again
- "Command timed out" or similar harness error
- Agent reports the command "hung" or "didn't respond"

**Root cause:** These commands poll until the Poof AI finishes, which can take 5-15+ minutes. Most tool harnesses default to 2-minute timeouts. The CLI itself enforces an internal 30-minute poll deadline on build/iterate/verify, so if your shell can hold the process open long enough, the CLI will wait for the AI to finish.

**Fix:** Set an extended timeout or run in background. See the timeout table in [SKILL.md](../SKILL.md) for recommended values per command.

| Command type | Recommended approach |
|-------------|---------------------|
| `poof build`, `poof iterate`, `poof verify`, `poof ship` | `run_in_background: true` (required — these can exceed the 10-min max timeout) |
| `poof security scan`, `poof deploy *` | `timeout: 600000` |
| All other commands | `timeout: 120000` (default is fine) |

**Example (Claude Code):**
```
# Background — best for build/iterate/ship (may exceed 10 min)
Bash tool: command="poof build -m '...'", run_in_background=true

# Extended timeout — for security scan, deploy
Bash tool: command="poof security scan -p <id>", timeout=600000
```

## CLI Version & Update Errors

| Symptom | Cause | Fix |
|---------|-------|-----|
| CLI prints `A new poof version is available` | The automatic daily update check found a newer GitHub release | Run `poof update --check` to inspect it. Update with `poof update`, or use `brew upgrade poofdotnew/tap/poof` for Homebrew-managed installs |
| Update notice appears where output must be parseable | Human text-mode command ran in an interactive terminal | Use `--json`, `--quiet`, `--no-update-check`, or set `POOF_NO_UPDATE_CHECK=1` |
| `poof update` says the binary is managed by Homebrew | The current executable resolves into a Homebrew Cellar path | Run `brew upgrade poofdotnew/tap/poof` |
| `poof update` refuses a `dev` or dirty build | The current version is not comparable to a release tag | Use `poof update --force` only when you intentionally want to replace the current binary with the latest release |

## Agent Recovery Patterns

### Stuck Build Recovery
```bash
poof chat cancel -p <id>
poof iterate -p <id> -m "The previous build got stuck. Review current state and continue."
```

### Stale Context Recovery
```bash
poof chat clear -p <id>
poof iterate -p <id> -m "Start a fresh AI context. Inspect the current project files, then apply this specific fix: ..."
```

### Credit Check Before Workflow
```bash
TOTAL=$(poof credits balance --json | jq '.credits.total')
if [ "$TOTAL" -lt 5 ]; then
  echo "Insufficient credits: ${TOTAL} remaining, need 5. Top up or wait for daily reset."
  exit 1
fi
```
