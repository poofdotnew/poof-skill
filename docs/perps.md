# Phoenix Perps Integration

Phoenix.trade's perp markets, wired through Poof so agents can register, fund, trade, and monitor positions via plain `poof data` calls. This doc is the one-stop for everything — markets, the full read surface, the write collections, position lookup patterns, and the math for converting lots to dollars.

For the general CLI shape, see [api-reference.md](api-reference.md). For composing writes atomically with guards, see [set-many.md](set-many.md).

## Contents
- [What this gives you](#what-this-gives-you)
- [Markets](#markets)
- [Lifecycle](#lifecycle)
- [Write collections](#write-collections)
- [Read queries](#read-queries)
- [Looking up all your positions](#looking-up-all-your-positions)
- [Converting lots to dollars](#converting-lots-to-dollars)
- [Composing with guards](#composing-with-guards)
- [Gotchas](#gotchas)

## What this gives you

Five onchain-action collections (`PhoenixRegister`, `PhoenixFund`, `PhoenixLong`, `PhoenixShort`, `PhoenixClose`) + one shared read-only collection (`commonQueries`) exposing seven reads from `@PhoenixPerpsPlugin`. All writes are user-scoped (`user/$userId/Phoenix*/...`) and pass through to Phoenix's mainnet program via CPI — the Poof service co-signs the CPI attestation, the caller's wallet pays the fees.

**The policy does not restrict which markets you can trade on.** `market` is just an address field on PhoenixLong/Short/Close. Any valid Phoenix perp market address works — new markets Phoenix adds don't require a policy redeploy. The plugin is a pass-through; market validation happens on Phoenix's program.

## Markets

Three canonical markets as of this writing. These are not policy constants — they're just addresses you can paste into any `market` field:

| Symbol | Market pubkey | Base lot decimals |
|---|---|---|
| SOL-PERP | `71Si24E4uc3oCaPbPZTozC1ptSNNqygjjebxSmErSsC2` | 6 |
| BTC-PERP | `AXFz1MuzMUBHi5UKJuK3FDCQ73o3rSzubGU2mPr4LLU7` | 6 |
| ETH-PERP | `9u7aqptdRFbsnnoHtjK13E5JkeM14EW5fAKTRPidVF88` | 6 |

If Phoenix lists more markets, look up the pubkey on [Phoenix](https://phoenix.trade) and use it directly. The plugin doesn't care which market you pass; the instruction is generic over market address.

## Lifecycle

A trader's journey, in the order an agent executes it:

```
PhoenixRegister  ──►  PhoenixFund  ──►  PhoenixLong / PhoenixShort  ──►  PhoenixClose
   (one-shot)        (USDC → margin)       (open position)              (reduce-only)
```

- **PhoenixRegister** creates the trader PDA for cross-margin. Required before anything else — unregistered traders get "trader account not found" on every downstream op. Idempotent at the plugin level: re-registering is a no-op but costs a tx. Poll `commonQueries.isRegistered` first.
- **PhoenixFund** bridges USDC to PhUSD and deposits into margin in one atomic hook. `amt` is in 6-dec USDC base units — convert human amounts client-side (`100 USDC = 100_000_000`).
- **PhoenixLong / PhoenixShort** place market orders. No slippage param — Phoenix market orders eat the book. If that's a concern, split into smaller `baseLots` and pace.
- **PhoenixClose** is reduce-only. `side: 1` (Ask) closes a long; `side: 0` (Bid) closes a short. Wrong side triggers a Phoenix ReduceOnly error. Always read `commonQueries.positionSize` first and pick `side = (size > 0 ? 1 : 0)`.

## Write collections

All take `user/$userId/<Collection>/$id` paths. The `$userId` segment has to equal `@user.address` — callers can only write their own collection entries.

### PhoenixRegister

```bash
poof data set -p <id> --path "user/<addr>/PhoenixRegister/reg-1" --data '{}'
```

No fields. The hook calls `@PhoenixPerpsPlugin.registerTrader(@user.address)`.

### PhoenixFund

```bash
poof data set -p <id> --path "user/<addr>/PhoenixFund/fund-1" \
  --data '{"amt": 100000000}'   # 100 USDC
```

Fields: `amt: UInt!` (6-dec USDC). Hook chains `emberDeposit` (USDC → PhUSD) + `depositFunds` (PhUSD → margin).

### PhoenixLong / PhoenixShort

```bash
# Long 0.1 SOL perp (baseLots = 0.1 * 10^6 = 100_000)
poof data set -p <id> --path "user/<addr>/PhoenixLong/long-1" \
  --data '{
    "market": "71Si24E4uc3oCaPbPZTozC1ptSNNqygjjebxSmErSsC2",
    "baseLots": 100000
  }'

# Short 0.1 SOL perp
poof data set -p <id> --path "user/<addr>/PhoenixShort/short-1" \
  --data '{"market": "71Si...", "baseLots": 100000}'
```

Fields: `market: Address!`, `baseLots: UInt!`. `baseLots = humanAmount × 10^baseLotsDecimals`.

### PhoenixClose

```bash
# Close a long: side=1 (Ask)
poof data set -p <id> --path "user/<addr>/PhoenixClose/close-1" \
  --data '{
    "market": "71Si...",
    "baseLots": 100000,
    "side": 1
  }'
```

Fields: `market: Address!`, `baseLots: UInt!`, `side: UInt!` (0 = Bid / close-short, 1 = Ask / close-long).

## Read queries

Live on the shared `commonQueries/$queryId` collection (public read, no `$userId` scope — pass the target wallet as a query arg). All plugin reads are **off-chain only** — they can't appear in onchain rule predicates, only as query expressions.

```bash
# Registration check
poof data query -p <id> --path 'commonQueries/x' --name isRegistered \
  --args '{"authority":"<addr>"}'
# → true | false

# Collateral (6-dec PhUSD base units)
poof data query -p <id> --path 'commonQueries/x' --name collateralBalance \
  --args '{"authority":"<addr>"}'

# Mark-to-market portfolio value (6-dec PhUSD)
poof data query -p <id> --path 'commonQueries/x' --name portfolioValue \
  --args '{"authority":"<addr>"}'

# Unrealized PnL (signed, 6-dec PhUSD)
poof data query -p <id> --path 'commonQueries/x' --name unrealizedPnl \
  --args '{"authority":"<addr>"}'

# Position size for a specific market (signed, in base lots; + long / − short / 0 none)
poof data query -p <id> --path 'commonQueries/x' --name positionSize \
  --args '{"authority":"<addr>","market":"71Si..."}'

# Has-position boolean for a specific market
poof data query -p <id> --path 'commonQueries/x' --name hasPosition \
  --args '{"authority":"<addr>","market":"71Si..."}'

# Mark price for a market (6-dec PhUSD, no authority needed — the plugin
# derives it from a reference trader's on-chain state)
poof data query -p <id> --path 'commonQueries/x' --name markPrice \
  --args '{"market":"71Si..."}'
```

The `commonQueries/x` path is arbitrary — the `$queryId` path segment doesn't feed any predicate, it just has to be present. Any stable string works; pick one and reuse it.

## Looking up all your positions

**There is no `getAllPositions(authority)` plugin call.** The way to enumerate is to ask `hasPosition(authority, market)` per market you care about. For the three canonical markets:

```bash
for m in 71Si24E4uc3oCaPbPZTozC1ptSNNqygjjebxSmErSsC2 \
         AXFz1MuzMUBHi5UKJuK3FDCQ73o3rSzubGU2mPr4LLU7 \
         9u7aqptdRFbsnnoHtjK13E5JkeM14EW5fAKTRPidVF88; do
  has=$(poof data query -p "$ID" --path 'commonQueries/x' --name hasPosition \
          --args "{\"authority\":\"$ADDR\",\"market\":\"$m\"}" --json \
        | jq -r .result)
  if [[ "$has" == "true" ]]; then
    size=$(poof data query -p "$ID" --path 'commonQueries/x' --name positionSize \
             --args "{\"authority\":\"$ADDR\",\"market\":\"$m\"}" --json \
           | jq -r .result)
    echo "$m → size=$size"
  fi
done
```

For a dashboard "am I healthy?" view, one `portfolioValue` + one `collateralBalance` + one `unrealizedPnl` call gives you the cross-margin summary without touching per-market detail.

## Converting lots to dollars

Two scales are in play:

- `positionSize` is in *base lots* for that market — integer, needs dividing by `10^baseLotsDecimals` (6 for the canonical markets) to get units of the base asset.
- `markPrice`, `portfolioValue`, `collateralBalance`, `unrealizedPnl` are in *PhUSD base units* — 6 decimals.

To get a position's dollar value (notional, at current mark):

```
notionalUSD = (positionSize * markPrice) / 10^(baseLotsDecimals + 6)
```

Worked example, SOL-PERP, 100_000 base lots long, mark price 85_000_000 (= $85.00):

```
notional = (100_000 * 85_000_000) / 10^(6 + 6)
         = 8.5e12 / 1e12
         = 8.5  →  $8.50
```

Agents should do this arithmetic client-side (after the two query calls) rather than trying to embed it in a query expression — plugin-call arithmetic inside the DSL is unreliable when types disagree.

## Composing with guards

Perps trades compose with Poof's guard primitives exactly like any other onchain action. Reject-before-fee patterns:

```bash
# Long only if SOL mark price is above some floor (sanity check against
# stale data / manipulated oracles — setMany rolls back if PriceGuard trips)
cat > long.json <<EOF
[
  {"path":"user/<addr>/PhoenixLong/long-1",
   "document":{"market":"71Si...","baseLots":100000}},
  {"path":"user/<addr>/PriceGuard/long-1-floor",
   "document":{"priceFeed":"SOL","op":"gte","threshold":70}}
]
EOF
poof data set-many -p <id> --from-json long.json

# Close only if position still exists (avoid sending reduce-only into an
# already-closed slot, which produces a cryptic Phoenix error)
# ... (use PhoenixClose + a RateLimit primitive to enforce 1/minute if an
# agent loop might double-send)
```

The `PriceGuard` primitive (see [set-many.md](set-many.md#poll-then-submit-with-simulate-queries)) is currently broken at the plugin level; use `commonQueries.markPrice` as a polling check *before* composing if you need a price gate.

## Gotchas

- **Read functions are off-chain only.** You can't use `@PhoenixPerpsPlugin.hasPosition(...)` inside an onchain rule, only in a query expression. Agents poll commonQueries client-side, then issue an unguarded write.
- **`baseLots` is per-market.** For the canonical markets it's 10^6 base units per 1.0 base asset, but new markets may differ. Always confirm from Phoenix's market metadata for anything outside the canonical set.
- **Market orders have no slippage parameter.** Phoenix eats the book. If that matters, split into smaller `baseLots` over time.
- **PhoenixClose `side` is a direction, not a fill side.** `1 = Ask` closes a long; `0 = Bid` closes a short. Read `positionSize` first, pick side by its sign.
- **Register is not idempotent at the tx level.** Re-registering succeeds but costs a tx. Gate on `isRegistered` before calling.
- **Funding uses PhUSD bridge internally.** The `PhoenixFund` hook handles the USDC → PhUSD conversion for you; don't try to call `depositFunds` directly on USDC.
- **No subaccount index support in `commonQueries`.** All reads use cross-margin (subaccount 0). Isolated subaccount reads would need a new commonQueries entry.
- **Non-Phoenix market addresses fail on-chain, not at the policy.** Since the policy no longer whitelists markets, a bad address is an on-chain revert (Phoenix CPI), which costs a fee via `--skip-preflight` or is silently rejected in preflight. Validate the market address before submitting.

## Related

- [set-many.md](set-many.md) — atomic bundles + the composition patterns that wrap Phoenix writes with guards
- [agents.md](agents.md) — the general agent loop; perps-specific examples live here
- [api-reference.md](api-reference.md) — full `poof data` flag reference
