# Deployment

Poof projects have four environments. Deploy to any of them using the `poof` CLI.

## Contents
- [Environments](#environments)
- [Pre-Deploy Checks](#pre-deploy-checks)
- [Publishing](#publishing)
- [Code Downloads](#code-downloads)
- [Custom Domains](#custom-domains)
- [Authentication on Deployed Apps](#authentication-on-deployed-apps)

## Environments

Poof has four distinct environments. Understanding the difference is critical — especially between Draft and Preview.

| Environment | Also called | Blockchain | URL pattern | Cost |
|-------------|------------|-----------|-------------|------|
| **Draft** | Poofnet | **Simulated** (fake tokens, free) | `{slug}-preview.poof.new` | Free |
| **Preview** | Mainnet Preview | **Real mainnet** (real tokens) | `{slug}-mainnet-preview.poof.new` | Any payment |
| **Production** | Live | **Real mainnet** (real tokens) | `*.poof.new` or custom domain | Any payment |
| **Mobile** | — | **Real mainnet** (real tokens) | Seeker / iOS / Android | Any payment |

### Draft (Poofnet) vs Preview (Mainnet Preview)

This is the most common source of confusion:

- **Draft / Poofnet** — Automatically available for every project, no deployment needed. Uses a **simulated blockchain** — all token transfers, staking, swaps etc. use fake/test tokens with zero real cost. Great for rapid iteration and testing. Token balances are simulated and won't reflect real amounts.

- **Preview / Mainnet Preview** — Requires explicit deployment. Uses the **real Solana mainnet** — transactions cost real SOL, tokens are real. This is your staging environment to test with real blockchain behavior before going to production.

**Key rule:** If you're just building and iterating, you're on Draft (Poofnet). Token balances are fake. When you deploy to Preview or Production, tokens become real.

Draft is always available — no deployment needed. Preview and production require that the user has **completed at least one credit purchase** via x402.

> **How it works:** The system calls `hasUserEverPaid()` — if the wallet has any completed payment record, deployment is unlocked permanently. If `poof deploy check` returns `no_membership`, buy credits via `poof credits topup` ($15 minimum for 50 credits) — this both provides AI credits and unlocks deployment.

## Pre-Deploy Checks

### Security Scan

`poof ship` initiates a security scan automatically before deploying and waits for it to complete before proceeding to the eligibility check and deployment. The CLI prints "Security scan finished" when the scan completes (neutral — it ran to completion), then checks eligibility.

**Only `critical` findings are a hard block.** `high`, `medium`, and `low` findings are reported but do not prevent the eligibility check from passing. The skill's default posture is to still flag high-severity findings to the user and have them decide whether to deploy — but the rule that Poof actually enforces is "no critical findings." If a scan returns e.g. 2 high + 0 critical, `poof deploy check` returns eligible and `poof ship` proceeds.

If the scan does find a critical, the CLI prints a warning and the eligibility check blocks deployment with a clear error. If both pass, it prints "Security scan passed. Eligible for deployment." To run a scan manually:

```bash
poof security scan -p <projectId>
```

### Eligibility Check

`poof ship` checks eligibility automatically before deploying. To check manually:

```bash
poof deploy check -p <projectId>
# Returns: payment status, security review status, project readiness
```

## Publishing

Deploy to preview (recommended first):

```bash
poof ship -p <projectId> --target preview
```

Or deploy to a specific environment directly:

```bash
# Deploy to preview
poof deploy preview -p <projectId>

# Deploy to production
poof deploy production -p <projectId> --yes

# Deploy to mobile (all three flags are required)
poof deploy mobile -p <projectId> --platform seeker --app-name "My App" --app-icon-url "https://..."
```

> **Note:** Mobile deploys require `--platform`, `--app-name`, and `--app-icon-url`. Platform values: `seeker`, `ios`, `android`.

The CLI automatically waits for the deploy task to complete before returning. If you need to check task progress manually:

```bash
# List all tasks for the project
poof task list -p <projectId>

# Get status of a specific task
poof task get <taskId> -p <projectId>
```

### Deployment Flow

1. **Deploy to preview first** — test with real mainnet behavior
2. **Verify on preview** — check functionality, transactions, UI
3. **Deploy to production** — when preview looks good
4. **Deploy to mobile** (optional) — requires `--platform`, `--app-name`, and `--app-icon-url`

## Static Frontend Deploy

If you're building your frontend outside of Poof (e.g. with `--mode backend,policy` or `--mode policy`) and want to deploy it to your Poof project, use `poof deploy static`. This uploads a pre-built `tar.gz` of your dist folder and hosts it on Poof alongside your backend.

```bash
poof deploy static -p <id> --archive dist.tar.gz
```

See [static-deploy.md](static-deploy.md) for the full guide, API reference, and examples.

## Code Downloads

Export your project's source code:

```bash
# 1. Initiate download (returns a taskId)
poof deploy download -p <projectId>

# 2. Check task status (poll until 'completed')
poof task get <taskId> -p <projectId>

# 3. Get signed download URL (expires in 5 minutes)
poof deploy download-url -p <projectId> --task <taskId>
# Note: download-url requires --task flag (not positional)
```

Requires at least one completed credit purchase.

## Custom Domains

After deploying to production, you can add custom domains:

```bash
# List existing domains
poof domain list -p <projectId>

# Add a custom domain
poof domain add myapp.com -p <projectId> --default
```

After adding a domain, configure your DNS to point to Poof's servers. The response includes the required DNS records.

## Authentication on Deployed Apps

Deployed apps need wallet authentication for users:

- **Phantom Connect (recommended)** — self-service, supports all Solana wallets + social login, free, works on custom domains immediately
- **Privy** — works on `poof.new` subdomains automatically; custom domains require contacting the Poof team
