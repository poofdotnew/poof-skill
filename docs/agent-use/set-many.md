# setMany — Atomic Multi-Document Writes

`setMany` packs multiple writes into one atomic transaction. Any rule, hook, or on-chain check that fails rejects the whole bundle; nothing partial is applied. That's what turns Poof's collection primitives into a composable toolkit — you compose a write with guards (BalanceCheck, TimeWindow, PriceGuard, etc.) and trust the transaction semantics to roll everything back on any failure.

**From the agent's perspective, everything here runs through `poof data ...` CLI commands.** The `@pooflabs/web` / `@pooflabs/server` SDKs are what the generated *frontend and backend code in the deployed app* use at runtime — agents and dev tools should drive the CLI.

## Project ID vs appId — and two ways to target

Before the examples, the thing to understand: a Poof **project ID** (what most CLI commands take via `-p <id>`) is your handle on the project you build and deploy. A Tarobase **appId** is the data-plane address of a *specific deployed policy instance*. **Each project has multiple appIds — one per environment** (draft = off-chain Poofnet, preview = real mainnet preview, production = real mainnet production), so the instances stay cleanly separated.

`poof data` takes either:

- **Project-based mode** — `-p <project-id> -e draft|preview|production`. The CLI resolves the right appId from the project's `connectionInfo`. Requires that you own (or have access to) the project.
- **Shared-appid mode** — `--app-id <id> --chain offchain|mainnet`. Talks to an appId directly. Useful when pointing at a *shared primitives library* someone else has deployed — no Poof-project access to it is needed, auth is still your own wallet, and any fees are paid by your wallet on your own writes. The user-scoped path convention (`user/$userId/...`) is what keeps callers safely sandboxed inside a shared appId: the rules pin each write to the caller's `@user.address`, so one wallet can't touch another's data.

List a project's appIds per env with:

```bash
poof data app-ids -p <project-id>
# → project <project-id>
#     draft      appId=... chain=offchain
#     preview    appId=... chain=mainnet
#     production appId=... chain=mainnet
```

All `poof data` subcommands below accept either `-p <id> -e <env>` or `--app-id <id> --chain <chain>`. The two are mutually exclusive — mixing them (e.g. `-e preview` with `--app-id`) errors out.

## Using `poof data set-many`

```bash
# Put your bundle in a JSON file (or stream via /dev/stdin).
cat > bundle.json <<'EOF'
[
  { "path": "user/<addr>/TokenTransfer/tt-1",
    "document": {"source":"<addr>","destination":"<peer>","mint":"<usdc>","amount":50} },
  { "path": "user/<addr>/BalanceCheck/bc-1",
    "document": {"mint":"<usdc>","op":"gte","threshold":10} }
]
EOF

# Project-based — targets whichever env you pick; default is draft
poof data set-many -p <project-id> --from-json bundle.json
# → ✓ submitted 2 document(s) on offchain (txid=sim_...)

# Or shared-appid — point at an appId directly. 69bcffc78d4b88997d0ed01a
# is the canonical shared generic-onchain primitives library on mainnet.
poof data set-many --app-id 69bcffc78d4b88997d0ed01a --chain mainnet --from-json bundle.json
```

The payload is either a bare array or `{"documents":[...]}` — both work. Each entry is `{path, document}`.

Targeting (project-based mode):
- `-e draft` (default) — Poofnet, simulated chain, safe for testing
- `-e preview` — mainnet preview, real tokens
- `-e production` — mainnet production, real tokens

Targeting (shared-appid mode):
- `--chain offchain` — talks to a draft/Poofnet appId
- `--chain mainnet` — talks to a real Solana mainnet appId (fees are real)

All entries in one bundle must be the same kind (all on-chain OR all off-chain). Mixed kinds are rejected.

## Single-doc writes, reads, queries

```bash
# Write one doc
poof data set -p <id> --path "memories/<addr>" --data '{"content":"hi"}'

# Read it back
poof data get -p <id> --path "memories/<addr>"

# Batch read
echo '["memories/<a>","memories/<b>"]' | poof data get-many -p <id> --from-json /dev/stdin

# Run a policy query
poof data query -p <id> --name getSolBalance --args '{"address":"<addr>"}'
```

## When to reach for setMany

Use setMany when correctness depends on two or more writes happening together:

- **Transfer + post-condition guard** — "transfer, but only if my balance stays ≥ floor afterward." A standalone read-check-write is a TOCTOU race; setMany evaluates the guard after the transfer within the same tx.
- **Swap + oracle price check** — slippage protection beyond what the plugin enforces.
- **Escrow create + initial fund** — avoids a half-initialized-escrow state.
- **Gated action** — `AllowlistCheck` + the gated write together, both succeed or neither.
- **Multi-leg DeFi** — deposit then lock, or withdraw then distribute.

If you only need one write and the condition is already expressible as a `rules.create` predicate, plain `poof data set` is enough — the rule gates it. Reach for setMany when the check is dynamic (balance after, oracle price now, cross-doc state).

## How rejection works

Assertions don't `throw` on Poof — **there's no `@assert` or `@require` primitive**. The gate is the collection's `rules.create` (or `.update` / `.delete`) expression: when it evaluates to `false`, the whole Solana tx is rejected. Rollback is inherent because it's one tx — no compensating deletes. Hooks are for side effects only; plugin calls chain with `&&` so a falsy guard short-circuits subsequent effects. Don't try to throw from a hook — write the predicate in the rule.

Where rejection actually surfaces depends on the collection kind:

| Collection kind | Surface |
|---|---|
| Off-chain, non-passthrough (`memories/$userId`) | Immediate `403` at REST with a full trace of the failed predicate; `poof data set` exits non-zero. |
| On-chain passthrough (`BalanceCheck`, `TokenTransfer`, etc.), **draft/poofnet** | Build-transaction step returns `202` with a tx payload; the draft RPC evaluates the rule server-side and surfaces failures as `rpc error -32603: User ... does not have access to create item at path ...`. |
| On-chain passthrough, **real mainnet (preview / production)** | The rule is baked into the Solana program's `set_documents` instruction and evaluated on-chain. Failing rules produce `AnchorError UnauthorizedCreate (6003) → custom program error 0x1773`, surfaced by Helius preflight simulation as `rpc error -32002: Transaction simulation failed: Error processing Instruction 0: custom program error: 0x1773`. The full rule-failure reason (including the offending path and submitted KV-pairs) is in the program `logMessages`. Preflight rejection means **the tx never lands on-chain and no fee is charged.** |
| On-chain non-passthrough (`Escrow`) | Same as passthrough at the corresponding env. |

So from an agent's perspective: a non-zero exit code + "does not have access" in the error = rule rejected, nothing applied. That's the signal you're looking for on both the `set` and the `set-many` paths.

Other failure sources in the same flow: insufficient on-chain balance (plugin runtime), RPC timeout (safe to resubmit; tx is idempotent by signature).

## Poll-then-submit with simulate queries

Every guard collection (BalanceCheck, TimeWindow, NftOwnershipCheck, AllowlistOnchain/OffchainCheck) exposes a `simulate` query that re-implements its create-rule predicate read-only. Agents can poll the simulate cheaply and only spend a Solana tx when it'd succeed:

```bash
# Would a BalanceCheck pass right now? (cheap, no tx)
poof data query -p <id> \
  --path "user/<addr>/BalanceCheck/any" --name simulate \
  --args '{"mint":"<usdc>","op":"gte","threshold":100}'
# → true | false

# Once true, submit the real bundle (setMany + guard)
poof data set-many -p <id> --from-json bundle.json
```

Agent loop shape:

```bash
# Watch "SOL < $100" as a trigger for a queued transfer
while :; do
  if [[ "$(poof data query -p $P --path user/$ME/PriceGuard/any --name simulate \
           --args '{"priceFeed":"SOL","op":"lt","threshold":100}' --json | jq -r .result)" == "true" ]]; then
    poof data set-many -p $P --from-json queued-transfer.json
    break
  fi
  sleep 30
done
```

The simulate query and the create rule share the same predicate, so a `true` from the simulate is a strong signal (not a guarantee — the state could change between poll and submit) that the real write would pass. Always still expect a potential rejection on the submit.

**Args convention:** inside a query expression, named `queryArgs` are bound to `@newData.<field>` (same as rules). `@user.address`, `@time.now`, plugin calls, and `get(/path)` all work. `$pathParam` tokens resolve to the path segments of the `--path` value, not to queryArgs.

**PriceGuard is currently broken.** `@PriceFeedPlugin.getPriceFeed(...)` returns a decimal string, not a UInt, so its create rule and simulate query can't evaluate the comparison. Schema is preserved with an always-`false` rule so callers get a clear rejection instead of a cryptic type error. Re-enable once the plugin returns numeric.

## Composition patterns

### Transfer with post-condition balance floor

```bash
cat > floor.json <<EOF
[
  {"path":"user/<addr>/TokenTransfer/tt-1",
   "document":{"source":"<addr>","destination":"<peer>","mint":"<usdc>","amount":50}},
  {"path":"user/<addr>/BalanceCheck/tt-1-floor",
   "document":{"mint":"<usdc>","op":"gte","threshold":10}}
]
EOF
poof data set-many -p <id> --from-json floor.json
```

The BalanceCheck is evaluated against post-bundle state; you can't race it.

### Swap with oracle-price guard

```bash
[{"path":"user/<addr>/MeteoraSwap/sw-1","document":{...}},
 {"path":"user/<addr>/PriceGuard/sw-1-guard",
  "document":{"priceFeed":"SOL","op":"gte","threshold":80}}]
```

### Allowlist-gated action

```bash
[{"path":"user/<addr>/AllowlistOnchainCheck/drop-v1","document":{"listId":"drop-v1"}},
 {"path":"user/<addr>/NftTransfer/nft-1","document":{...}}]
```

## Running `poof data` alongside other CLI commands

Safe to run concurrently across terminals or shell-backgrounded jobs — the CLI was hardened for this. Some specifics worth knowing:

- **`poof data` (any appid) + `poof iterate|build|ship` (your own project)** — independent. Different APIs (`api.tarobase.com` vs `api.poof.new`), different on-disk caches (`~/.poof/tarobase-sessions.json` vs `~/.poof/tokens.json`). Running a long `poof iterate` while firing `poof data set` against a shared appid works with no interference.
- **Two `poof data` calls against different appids** — safe. Sessions are cached per (appId, wallet), so each appid has its own cache entry and the two processes don't compete for the same slot. The session-cache file is written atomically (temp + rename), so a mid-write read by one process can't see a torn JSON from another's write.
- **Two `poof data` calls against the *same* appid** — also safe. Both processes may notice "no cached session" at startup and each do a login (that's a small waste, not a correctness issue). The atomic write means neither process can corrupt the other's cache entry.
- **Keypair is read-only at runtime.** All concurrent invocations use the same `SOLANA_PRIVATE_KEY` from `.env`; no write contention there.
- **Don't run `poof config set` while other CLI commands are in flight.** That's the one writable non-atomic state, and it's only touched by explicit `config set` commands — nothing else writes to `~/.poof/config.yaml`.

## Gotchas

- **Same kind only** — on-chain or off-chain in one bundle, not both.
- **Solana tx size (~1232 bytes)** caps how many on-chain ops fit. Short field names matter on high-fanout collections. Splitting loses cross-split atomicity.
- **Passthrough guards still cost bytes** — each guard produces Solana instructions.
- **Use `getAfter` inside rules that need post-bundle state**; plain `get` reads pre-bundle.
- **No ternary, no switch/case.** Multi-branch dispatch in a rule is `(cond && A) || (!cond && B)` chained — that's how one BalanceCheck/PriceGuard supports `gte | lte | eq | gt | lt` on a string `op` field.
- **No string-concat `+` for path building.** `+` is arithmetic only. Embed `@newData.<field>`, `@user.address`, `$pathParam` directly: `get(/Allowlist/@newData.listId/member/@user.address)`.
- **`@NFTPlugin` has no ownership query.** Use `@TokenPlugin.getBalance(owner, mint) >= 1` for SPL-based NFTs; Metaplex Core assets aren't hook-checkable.
- **On-chain rules can't read off-chain data.** If an on-chain gate depends on an allowlist, the allowlist must be on-chain too.
- **Distinct ids per bundle entry** — path collisions within a bundle fail rule evaluation.

## What the CLI does under the hood

For debugging, this is the raw flow the CLI orchestrates — you don't need it directly:

1. `PUT /items` with `{documents:[{destinationPath, document}...]}` on Poof's data-plane API. Returns `offchainTransaction` (draft) or `transactions` (mainnet).
2. **Draft:** SHA-256-hash the `tx.message` JSON, base64-wrap the signed-tx envelope, `POST /app/{appId}/rpc` with `sendTransaction` to Poof's offchain RPC endpoint.
3. **Mainnet:** borsh-encode the Anchor `set_documents` args (app_id, documents, delete_paths, tx_data, simulate), build a VersionedTransaction with `preInstructions` + `set_documents`, fetch the lookup table contents from Solana RPC, resolve into `MessageV0`, sign with the wallet keypair, submit via Solana RPC. Returns the Solana transaction signature.

CPI co-signing (`txData` / server-`signedTransaction` responses) is the one path not yet covered by the CLI — very few collections produce it, and when they do the CLI errors with a clear message pointing at the `@pooflabs/server` SDK. All normal reads, writes, and setMany bundles go through the CLI end-to-end.

See [api-reference.md](../api-reference.md) for full CLI flags and JSON output shapes.

## Related

- [api-reference.md](../api-reference.md) — full `poof data` flag reference
- [database-sdk.md](../database-sdk.md) — the generated frontend/backend SDK used *inside* a deployed app
- [how-poof-works.md](../how-poof-works.md#passthrough-pattern) — passthrough collections, which most guards are
- [testing.md](../testing.md) — lifecycle-action `setMany` op for test files
