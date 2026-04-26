# Credits & Payments

## Contents
- [Credit System](#credit-system)
- [Paid Features](#paid-features)
- [x402 Credit Top-Up](#x402-credit-top-up)
- [Per-Project Credit Bank](#per-project-credit-bank)
- [Usage & Overuse Limits](#usage--overuse-limits)
- [AI Preferences](#ai-preferences)

## Important: How Payment Unlocks Features

Poof gates paid features (mainnet deployment, code downloads, custom domains) on whether the user has **ever completed any credit purchase**. There is no separate membership or subscription required.

| What you need | How to get it |
|---------------|---------------|
| **AI credits** (for chat messages, project builds) | Free daily allotment (~10/day), OR buy add-on credits via x402 USDC |
| **Paid features** (mainnet deploy, code downloads, custom domains) | Complete any credit purchase. Once you've purchased credits, paid features are permanently unlocked for that wallet |

**Key point:** A single x402 credit purchase unlocks both AI credits AND paid features (deployment, downloads, etc.). The eligibility check (via `poof deploy check`) passes as long as `hasUserEverPaid` is true for the wallet.

## Credit System

Poof uses credits for AI interactions (chat messages, builds). Check your balance before starting long workflows:

```bash
poof credits balance            # Human-readable summary
poof credits balance --json     # Machine-readable JSON output
```

### Credit Types

| Type | Source | Details |
|------|--------|---------|
| Daily | Free | ~10 credits, reset daily, available to all users |
| Subscription | (deprecated) | Still present in the API response as `subscription: { remaining, purchased }` but no longer actively used. Ignore for new integrations. |
| Add-on | x402 purchase | Bought with USDC, expire in 6 months |

### Credit Cost Estimates

| Operation | Approx. Credits |
|-----------|----------------|
| `poof build` (full build) | 1-2 |
| `poof iterate` / `poof chat send` (follow-up message) | 1 |
| Full build + test + polish cycle | 3-5 |

### Handling Credit Exhaustion

Always check credits before starting and between steps. If credits run out mid-build, the AI stops responding -- `poof chat send` calls will return errors and `poof chat active` may hang.

```bash
# Check before each major step -- need at least 3 for build + test + polish
TOTAL=$(poof credits balance --json | jq '.credits.total')
if [ "$TOTAL" -lt 3 ]; then
  echo "Insufficient credits: $TOTAL remaining, need 3. Buy more or wait for daily reset."
  exit 1
fi
```

## Paid Features

Paid features require **any completed credit purchase**:
- Preview and production deployments (mainnet)
- Code downloads
- Custom domains
- Advanced AI model preferences

The system checks `hasUserEverPaid()` -- if the wallet has any completed payment record, paid features are unlocked **permanently**.

### How to Unlock

Buy credits via `poof credits topup` (USDC on Solana, no browser needed). A single purchase ($15 minimum) creates a payment record and unlocks all paid features.

## x402 Credit Top-Up

Buy credits with USDC on Solana -- no browser needed:

```bash
poof credits topup               # Buy 1 package ($15 = 50 credits)
poof credits topup --quantity 5   # Buy 5 packages ($75 = 250 credits)
```

The CLI handles the entire x402 USDC payment flow automatically. Your wallet must have sufficient USDC on Solana mainnet.

> A completed x402 credit purchase also unlocks all paid features (mainnet deployment, code downloads, custom domains) since the system only requires that the user has ever completed any payment.

### Pricing

| Quantity | USDC | Credits | Expiry |
|----------|------|---------|--------|
| 1 package | $15 | 50 | 6 months |
| 5 packages | $75 | 250 | 6 months |
| 10 packages (max) | $150 | 500 | 6 months |

Any authenticated user can purchase. Payments are idempotent via transaction ID. A completed purchase also unlocks paid features (deployment, downloads, etc.).

### Troubleshooting x402 Payments

If `poof credits topup` fails, check:
1. Your wallet has sufficient USDC on Solana mainnet
2. You are authenticated (`poof auth login`)
3. Quantity is between 1 and 10

## Per-Project Credit Bank

Pre-fund a project's wallet so its runtime usage and AI chat draw from there before falling back to your account. Owner-only mutations; admins / collaborators read-only.

### Buckets

| Bucket     | Spendable on                              | Default? |
| ---------- | ----------------------------------------- | -------- |
| `combined` | Either runtime usage or AI chat           | Yes      |
| `usage`    | Runtime only: infra compute, storage, gas | No       |
| `chat`     | AI chat / Claude development only         | No       |

Drain order: purpose-specific bucket → `combined` → owner's account balance (unless that purpose is isolated). Each bucket splits into *withdrawable* (your deposits) and *reserved* (Poof-granted) pools — only withdrawable can be pulled back out.

### Commands

```bash
poof credits project status -p <id>

# Deposit pulls from paid (subscription + add-on) credits — never daily.
poof credits project deposit -p <id> --amount 50                  # combined
poof credits project deposit -p <id> --amount 100 --bucket usage

# Withdraw creates a fresh add-on payment record (6-month expiry).
poof credits project withdraw -p <id> --amount 30

# Isolation: when true, that purpose pauses on empty (no account-balance fallback).
poof credits project isolation -p <id> --usage true --chat false
```

### Picking a policy

- **Default (recommended).** Deposit `combined`, leave both isolations off. Wallet covers spend; account balance picks up after.
- **Hard cap.** Set `--usage true` (and/or `--chat true`) to pause instead of falling back.
- **Per-purpose accounting.** Use `usage` / `chat` buckets even with isolation off, just to track separately.

### Concurrency

One withdrawal per (user, project) at a time — concurrent calls get `402`. Deposits + reads are atomic. Negative balances are clamped at 0 so transient overdrafts don't reject legitimate operations.

### Common errors

| Error                                                  | Fix                                                                       |
| ------------------------------------------------------ | ------------------------------------------------------------------------- |
| `Insufficient paid credits to deposit N`               | `poof credits topup` first.                                               |
| `Project's <bucket> withdrawable bucket has X credits` | Re-run `status`, retry with a smaller amount.                             |
| `A withdrawal is already in progress for this project` | Wait — stale locks auto-clear after 5 minutes.                            |
| `Only the project owner can deposit/withdraw credits`  | Use the owner's wallet.                                                   |

## Usage & Overuse Limits

```bash
poof usage status -p <id>                  # month-to-date cost / pause state
poof usage limit  -p <id> --credits 50     # cap paid overage at N credits
poof usage limit  -p <id> --clear          # no cap (app pauses at free-tier end)
poof usage resume -p <id>                  # unblock a paused project
```

Key fields in `status`:

- `costCredits` / `freeCreditsApplied` / `chargedCredits` — spend / free-tier coverage / overage.
- `paidCreditsRemaining` — available to cover overage (your account balance plus the project wallet's `usage`-spendable portion when not isolated).
- `isBlocked` / `canResume` / `blockedReason` — pause state.
- `summaryStale` / `blockedStatusStale` — upstream pipeline failed; **don't act on the corresponding fields**.

### When the app is paused

| `blockedReason`        | Fix                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------ |
| `no_overuse_limit`     | `poof usage limit -p <id> --credits N` → `poof usage resume`.                         |
| `threshold_reached`    | Raise `--credits` → `poof usage resume`.                                              |
| `insufficient_credits` | `poof credits topup` or `poof credits project deposit` → `poof usage resume`.         |

`resume` rejects with 400 if preconditions aren't met — fix first, then retry.

## AI Preferences

Control the intelligence tier used for different AI operations:

```bash
poof preferences get                                    # View current preferences
poof preferences set mainChat=genius codingAgent=smart  # Update preferences (requires any credit purchase)
```

Available tiers: `average`, `smart`, `genius`.
