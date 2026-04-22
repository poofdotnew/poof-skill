# Guard primitives

Reusable predicate collections you compose with actions in a `poof data set-many` bundle. When a guard's `create` rule evaluates to `false`, the whole bundle rolls back — the action next to it never lands.

Two kinds of guards:

- **User-scoped asserting passthroughs** — write under `user/$userId/<Guard>/$id`. The write itself is the assertion; a failed rule aborts the containing setMany. These are the guards you put in a bundle.
- **Global config + per-user check** — for Allowlist / Escrow, the admin defines the list/escrow as a standalone collection and agents use the corresponding `...Check` / `...Withdraw` collection to assert membership or withdraw.

Every guard has a matching `simulate` query that re-runs its predicate read-only, so an agent can poll before submitting and avoid landing a known-failing bundle.

For how to compose these with actions, see [set-many.md](set-many.md). For the action catalog they gate, see [onchain-actions.md](onchain-actions.md).

## The simple user-scoped guards

| Guard | Fields | What the create rule asserts |
|---|---|---|
| `BalanceCheck` | `mint: Address`, `op: "gte"\|"lte"\|"eq"\|"gt"\|"lt"`, `threshold: UInt` | `TokenPlugin.getBalance(caller, mint) <op> threshold` (base units). |
| `TimeWindow` | `startTime, endTime: UInt` (unix seconds) | `startTime ≤ time.now ≤ endTime`. |
| `NftOwnershipCheck` | `mint: Address` | Caller holds ≥1 of `mint`. Uses `TokenPlugin.getBalance` since `@NFTPlugin` has no ownership query — works for SPL-based NFTs, not Metaplex Core assets. |
| `PriceGuard` | `priceFeed: String`, `op, threshold` | **Broken.** `@PriceFeedPlugin.getPriceFeed` returns a decimal string; the rule can't do the numeric compare. Rule is a permanent `false` — don't use until the plugin returns UInt. |

Simulate:

```bash
poof data query --app-id <appId> --chain mainnet \
  --path "user/<self>/BalanceCheck/any" --name simulate \
  --args '{"mint":"<usdc>","op":"gte","threshold":100000000}'
# → true | false
```

Compose in a bundle:

```json
[
  {"path":"user/<self>/TokenTransfer/tt-1",
   "document":{"source":"<self>","destination":"<peer>","mint":"<usdc>","amount":50000000}},
  {"path":"user/<self>/BalanceCheck/tt-1-floor",
   "document":{"mint":"<usdc>","op":"gte","threshold":10000000}}
]
```

## Allowlist (onchain trio)

For gating *onchain* actions by admin-controlled membership. On-chain rules can't read off-chain state, so the list has to be on-chain too.

| Collection | Who can write | Shape |
|---|---|---|
| `AllowlistOnchain/$listId` | admin defines | `name: String`, `admin: Address`. Rule: `newData.admin == user.address`. |
| `AllowlistOnchain/$listId/member/$memberAddress` | only that list's admin | `addedAt: UInt`. Existence-of-row = membership. |
| `user/$userId/AllowlistOnchainCheck/$id` | caller asserts their own membership | `listId: String`. Rule: `get(/AllowlistOnchain/<listId>/member/<caller>) != null`. |

Lifecycle:

```bash
# 1. Admin creates the list + adds members (one-time, per list)
poof data set-many --app-id <appId> --chain mainnet --from-json - <<'EOF'
[
  {"path":"AllowlistOnchain/vip","document":{"name":"VIPs","admin":"<admin-addr>"}},
  {"path":"AllowlistOnchain/vip/member/<alice>","document":{"addedAt":1730000000}},
  {"path":"AllowlistOnchain/vip/member/<bob>","document":{"addedAt":1730000000}}
]
EOF

# 2. Alice gates a transfer on being in the list
poof data set-many --app-id <appId> --chain mainnet --from-json - <<'EOF'
[
  {"path":"user/<alice>/TokenTransfer/t1","document":{...}},
  {"path":"user/<alice>/AllowlistOnchainCheck/t1-gate","document":{"listId":"vip"}}
]
EOF
```

## Allowlist (offchain trio)

Same shape but off-chain. Free (no rent). Can only gate *offchain* writes — onchain rules can't read offchain state.

| Collection |
|---|
| `AllowlistOffchain/$listId` |
| `AllowlistOffchain/$listId/member/$memberAddress` |
| `user/$userId/AllowlistOffchainCheck/$id` |

Use for gating things like `memories/$userId` writes or other offchain-only workflows.

## RateLimit (onchain + offchain counters)

User-scoped sliding-window counters.

| Collection | Notes |
|---|---|
| `user/$userId/RateLimitOnchainCounter/$key` | Onchain, pays rent. Create initializes `count=0, windowStart=now, windowSeconds, maxPerWindow`. Update rule enforces window reset and cap atomically: increments bump `count`; if `time.now - windowStart >= windowSeconds`, `count` resets. Updates beyond `maxPerWindow` within the window are rejected. |
| `user/$userId/RateLimitOffchainCounter/$key` | Offchain-only variant of the same. Free; gates offchain writes only. |

Typical use: one counter per (user, action-kind). Include the counter increment in the bundle with the action; if the cap is exceeded the whole bundle aborts.

```bash
# Initialize once (count=0, window=3600s, cap=10)
poof data set --app-id <appId> --chain mainnet \
  --path "user/<self>/RateLimitOnchainCounter/transfers" \
  --data '{"count":0,"windowStart":1730000000,"windowSeconds":3600,"maxPerWindow":10}'
```

## Escrow (onchain trio)

Simple single-recipient-set escrow. Depositor funds, lists allowed withdrawers; any allowed withdrawer drains the full balance.

| Collection | Who writes | Shape |
|---|---|---|
| `Escrow/$escrowId` | depositor creates + funds | `depositor: Address`, `amount: UInt`, `mint: Address`. Create hook moves `amount` of `mint` from depositor to a PDA vault. |
| `Escrow/$escrowId/allowedWithdrawer/$withdrawerAddress` | only the depositor | `addedAt: UInt`. |
| `user/$userId/EscrowWithdraw/$id` | allowed withdrawers | `escrowId: String`. Rule: `get(/Escrow/<id>/allowedWithdrawer/<caller>) != null` AND vault has a positive balance. Hook full-drains into the caller. |

```bash
# Depositor sets up in one atomic bundle
poof data set-many --app-id <appId> --chain mainnet --from-json - <<'EOF'
[
  {"path":"Escrow/payout-1","document":{"depositor":"<me>","amount":1000000,"mint":"<usdc>"}},
  {"path":"Escrow/payout-1/allowedWithdrawer/<alice>","document":{"addedAt":1730000000}}
]
EOF

# Alice withdraws
poof data set --app-id <appId> --chain mainnet \
  --path "user/<alice>/EscrowWithdraw/w-1" --data '{"escrowId":"payout-1"}'
```

## Gotchas

- **Onchain rules can't read offchain state.** If an onchain action needs to gate on an allowlist, the allowlist must be onchain.
- **Per-guard entry ids must be distinct within a bundle.** Path collisions abort rule evaluation.
- **Every guard adds Solana-instruction bytes.** Tx size is ~1232 bytes; stacking five guards on one action can push you over.
- **`simulate` results are a hint, not a guarantee.** State can change between poll and submit — always expect a potential rejection on the submit even after a `true` simulate.
- **PriceGuard is currently broken** (permanent `false` rule). Use `commonQueries.markPrice` from the Phoenix integration (see [perps.md](perps.md)) or a direct plugin query as a polling check outside the bundle if you need price-gated action.
