# Poof On-Chain Capability Catalog — "Can Poof express this?"

**This is the authoritative decision procedure. You MUST consult it before concluding Poof cannot express an on-chain behavior.** Almost every on-chain behavior an app needs is expressible as Poof collections + policy rules + plugin hooks (optionally composed atomically with `setMany`). Agents repeatedly *hallucinate* that Poof can't do something — escrow, treasuries, voting, perp trading, proportional payouts — when it can. That hallucination is the single most expensive failure mode in the pipeline.

## The hard rule

1. **Poof owns 100% of on-chain transaction/write behavior.** Every transfer, mint, burn, swap, trade, order, escrow, vault movement, settlement, payout, or program instruction goes through Poof policy/plugins/hooks.
2. **Custom Anchor / Rust Solana programs / raw `@solana/web3.js` / raw RPC are NEVER the answer.** Not as a fallback, not as an "escape hatch", not "just for this one instruction". There is no `Platform: anchor` opt-out.
3. **A genuine gap is escalated to *extend Poof*, never worked around.** If — after running the decision procedure below — no composition of Poof primitives expresses the behavior, you stop and emit `blocked:platform-poof-onchain-primitive-missing` with the exact missing primitive contract (action name, fields, target program, guard semantics, signer/actor boundary, expected postconditions). That escalation goes to the operator, who extends Poof's plugins/policy. It is **not** permission to hand-roll the behavior. Custom on-chain code is a stop, not a route-around.
4. **The presumption is expressibility.** Anything on-chain that is not on the short, closed "Genuinely cannot" list (below) is presumed expressible. The burden is on you to prove non-expressibility against this catalog *before* you may escalate — not to assume it.

## Behavior → Poof primitive lookup

If your requirement decomposes into any of these, it is expressible. Compose multiple rows with `setMany` for atomic multi-step behavior.

| Behavior you think needs custom on-chain code | How Poof expresses it |
|---|---|
| **Escrow / vault / treasury holding pooled funds** | `Escrow/$escrowId` on-chain trio (create → fund → release/withdraw), see [guards.md](guards.md); or the server-managed **project vault** `@constants.PROJECT_VAULT_ADDRESS` / `PROJECT_VAULT_PRIVATE_KEY`; or `@AccountPlugin` PDAs ([how-poof-works.md](../how-poof-works.md) Plugin Ecosystem) |
| **Equal-stake / fixed-amount / exact-fee enforcement** | A `rules.create` predicate on the funding collection: `@newData.amount == get(/rooms/$roomId).entryFee` (assertions live in policy rules) |
| **Voting / proposals / majority / quorum / governance** | Off-chain `proposals/$id` collection + `proposals/$id/votes/$voterAddress` subcollection (one row per voter = dedupe by path); the gated action's `rules.create` tallies via `get()` / `getAfter()`. No on-chain program needed for the vote itself |
| **Perp trade execution (open/close long/short, fund/withdraw margin)** | Poof Phoenix perps: `PhoenixRegister` / `PhoenixFund` / `PhoenixLong` / `PhoenixShort` / `PhoenixClose` / `PhoenixWithdraw` + `commonQueries` reads — the **complete** Phoenix perps lifecycle. See [perps.md](perps.md) |
| **Spot swap / DEX trade / liquidity** | `@DeFiPlugin` (Jupiter swaps, Meteora pools/CP-AMM), `MeteoraSwap`, see [onchain-actions.md](onchain-actions.md) |
| **Token launch with tradeable bonding curve** | `@DeFiPlugin.createMeteoraVirtualPool` (default) or `@PumpFunPlugin.createToken` (Pump.fun) |
| **Proportional / pro-rata / multi-party payout** | A release/dissolve collection whose hook chains multiple `@TokenPlugin.transfer(...)` with `&&`, or an atomic `setMany` bundle; amounts computed from on-chain state via `get()` |
| **Conditional / gated transfer (only if X)** | Guard primitive + action in one `setMany`: `BalanceCheck`, `TimeWindow`, `NftOwnershipCheck`, `AllowlistOn/OffchainCheck`, `RateLimit*`, `PriceGuard` — see [guards.md](guards.md) |
| **Allowlist / whitelist / invite-only** | `AllowlistOnchain/$listId` (+ `/member/$addr`) or `AllowlistOffchain` trio — admin defines, member asserts membership |
| **Rate limiting / cooldown / once-per-period** | `RateLimitOnchainCounter` / `RateLimitOffchainCounter` + check, see [guards.md](guards.md) |
| **Time-bounded action (window open/close, deadline)** | `TimeWindow` guard and/or `@time.now` in `rules.create` |
| **Admin / role / owner-only action** | Compare `@user.address` against an admin constant in `rules` (no built-in role system — this is the documented pattern) |
| **Randomness (games, lotteries, raffles, gambling)** | `@OraclePlugin` VRF |
| **Prediction market** | `@PredictionMarketPlugin` (LMSR/AMM) or `@DflowPlugin` (Kalshi) |
| **NFT mint / collection / buy / list / burn** | `@NFTPlugin`, `@TensorPlugin`, `NftTransfer/NftCreateCollection/NftBurn/TensorBuyNft/TensorListNft` |
| **Multi-step atomic on-chain workflow** | `setMany` — all-or-nothing per Solana tx, see [set-many.md](set-many.md) |
| **"On-chain program instruction X"** | Translate to: collection + `rules` (the assertion X enforces) + a plugin hook (the state change X makes). An Anchor instruction is a policy rule + hook in Poof terms |

## Decision procedure (run this before ever concluding "Poof can't")

1. **Decompose** the behavior into primitives: a *transfer*, an *escrow/vault hold*, a *gate/assertion*, a *payout*, a *trade*, a *mint/burn*, a *vote tally*, a *time/rate/allowlist constraint*. Almost everything is a composition of these.
2. **Look up each piece** in the table above and in [onchain-actions.md](onchain-actions.md), [guards.md](guards.md), [perps.md](perps.md), and the [how-poof-works.md](../how-poof-works.md) Plugin Ecosystem table.
3. **Express constraints as `rules.create` predicates.** "Only if", "exactly equal to", "before deadline", "majority reached", "caller is admin" — these are policy-rule expressions over `@newData`, `@user`, `@time`, `@constants`, and `get(...)` cross-reads. They are not reasons to leave Poof.
4. **Compose multi-step behavior with `setMany`** (atomic) rather than assuming you need a custom program to bundle instructions.
5. **Only if** no composition of steps 1–4 expresses it: it is a genuine gap. **Stop. Emit `blocked:platform-poof-onchain-primitive-missing`** with the full primitive contract. Do **not** author or request a custom Anchor/Solana program. The resolution is the operator extending Poof.

## Genuinely cannot (closed list — everything on-chain else is presumed expressible)

- ML/AI model *training* → external API via backend
- Video/audio processing → external service via backend
- Non-Solana blockchains → Solana only
- Native SOL staking → use liquid staking tokens via `@DeFiPlugin.swap`

If a requirement is not one of these four, **it is presumed expressible** — verify against this catalog and escalate the specific missing primitive if and only if you can show every row above fails. "I couldn't immediately see how" is not proof; it is the hallucination this document exists to stop.

## Why this matters

A product that wrongly concludes "Poof can't do escrow/voting/perps" and reaches for a custom Anchor program will: (a) violate the hard no-custom-on-chain rule, (b) need a toolchain the agent host doesn't have, (c) stall in QA for days, and (d) have been buildable in Poof the entire time. This has already happened (phoenix-chat: a full Anchor program for treasury + equal-stake + voting + Phoenix execution — *every piece of which is in the table above*). Consult this catalog first, every time.

## Related

- [onchain-actions.md](onchain-actions.md) — the full action collection catalog
- [guards.md](guards.md) — escrow, allowlist, time, rate, balance, price guard primitives
- [perps.md](perps.md) — complete Phoenix perps lifecycle
- [set-many.md](set-many.md) — atomic multi-step composition
- [../how-poof-works.md](../how-poof-works.md) — policy/rules/hooks/plugins architecture
