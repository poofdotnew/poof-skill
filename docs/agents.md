# Writing Agents That Live on Poof

This doc is the shape of an agent running continuously on top of a Poof project — read its own state, poll the world, decide, act atomically, verify. Everything is driven through `poof data …`; no SDK import needed.

For the atomicity story of the "act" step, see [set-many.md](set-many.md). For the CLI surface, see [api-reference.md](api-reference.md). Both targeting modes work for every command below: project-based (`-p <project-id> -e <env>`, agent owns the project) or shared-appid (`--app-id <id> --chain offchain|mainnet`, agent points at a shared primitives library someone else deployed). The examples below use `-p <id>` for brevity; swap in `--app-id <id> --chain <chain>` if you're running against a shared appid. See [set-many.md — Project ID vs appId](set-many.md#project-id-vs-appid--and-two-ways-to-target) for the distinction.

## Contents
- [The loop](#the-loop)
- [Reading your own state](#reading-your-own-state)
- [Polling the world](#polling-the-world)
- [Deciding](#deciding)
- [Acting atomically](#acting-atomically)
- [Verifying + recording](#verifying--recording)
- [Putting it together](#putting-it-together)

## The loop

```
┌─ read own state (memories/<self>, own counters, own positions)
│
├─ poll the world (balances, prices, on-chain reads, simulate queries)
│
├─ decide — is the precondition met? what action?
│
├─ act — setMany bundle with guards, atomic all-or-nothing
│
└─ verify — read back, update memories, loop
```

Every step above is a `poof data` call. No long-running process needed; the agent can be a cron entry, a webhook, a per-tick script.

## Reading your own state

The simplest stateful agent stores per-agent notes in `memories/$userId` (one document per wallet, keyed by the agent's own address, private by policy — only you can read or write). It's off-chain and free.

```bash
# Your own memory
poof data get -p <id> --path "memories/<self-addr>"
# → [{"id":"<self-addr>","content":"last check 1776726280, balance was 45 USDC", ...}]
```

If your agent manages per-user counters (rate limits, per-session state) or per-action history, those live under `user/$userId/<Collection>/$id`. Since the rule is `$userId == @user.address`, you can read/write anything under `user/<your-addr>/...` and nothing else:

```bash
# List your own TokenTransfer history
poof data get -p <id> --path "user/<self-addr>/TokenTransfer"

# Read your own RateLimit counter
poof data get -p <id> --path "user/<self-addr>/RateLimitOffchainCounter/my-key"
```

The user-scoped path prefix means `poof data get --path "user/<self-addr>"` gives you a cross-collection index of your own activity — one call returns everything your agent has done.

## Polling the world

For state not local to your wallet — market prices, on-chain balances, another user's allowlist membership — use policy queries. Most live at the global `queries/$queryId` collection and are cheap read-only calls:

```bash
poof data query -p <id> --name getSolPriceInUSD
# → "85.54"

poof data query -p <id> --name getSolBalance --args '{"address":"<some-addr>"}'
# → "97725734"
```

For guard primitives (BalanceCheck, TimeWindow, NftOwnershipCheck, AllowlistOnchainCheck, AllowlistOffchainCheck) and project-specific dashboards (`commonQueries/$queryId`), each exposes a `simulate` query you hit with `--path`:

```bash
# Would a BalanceCheck pass right now?
poof data query -p <id> \
  --path "user/<self-addr>/BalanceCheck/any" --name simulate \
  --args '{"mint":"<usdc>","op":"gte","threshold":100}'
# → true | false
```

See [set-many.md — Poll-then-submit](set-many.md#poll-then-submit-with-simulate-queries) for the full pattern.

## Deciding

Keep the decision logic in the agent's own code. Policy rules enforce the final "is this allowed," but *prioritization* (which action to take, in what order) is agent-local.

A common shape: an agent reads its own memory + runs a simulate → if simulate returns `true`, proceed; otherwise sleep and retry on the next tick. The memory acts as an idempotency marker ("did I already do this for this opportunity?") since policy rules can't dedupe on arbitrary criteria.

## Acting atomically

When you commit, use `poof data set-many` and bundle the action with its guards. Any guard's rule failure aborts the whole bundle — no partial state. See [set-many.md](set-many.md) for composition patterns.

```bash
cat > bundle.json <<'EOF'
[
  { "path": "user/<self>/TokenTransfer/<id>",
    "document": {"source":"<self>","destination":"<peer>","mint":"<usdc>","amount":50} },
  { "path": "user/<self>/BalanceCheck/<id>-floor",
    "document": {"mint":"<usdc>","op":"gte","threshold":10} }
]
EOF
poof data set-many -p <id> --from-json bundle.json
```

On draft (`-e draft`, default) the whole thing is simulated and free. On mainnet (`-e preview` or `-e production`) the CLI signs + submits a real Solana versioned transaction via Helius.

## Verifying + recording

After a write lands, record what you did in your own memory so the next tick can reason about it:

```bash
poof data set -p <id> --path "memories/<self>" --data "$(cat <<EOF
{"content":"last transfer $(date -u +%s) — 50 USDC to <peer>, tx sim_..."}
EOF
)"
```

For onchain actions, the `set-many` result carries a `transactionId` (simulated id on draft, Solana signature on mainnet). Persist it if you need an audit trail — or just query `user/<self>/TokenTransfer` on the next tick to see the live server-side record.

## Putting it together

A minimal peg-to-target agent — "keep my USDC balance at ≥ 100; if it dips, top up from a vault":

```bash
#!/usr/bin/env bash
set -euo pipefail
P=<project-id>
SELF=<agent-wallet>
USDC=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
VAULT=<vault-addr>

while :; do
  # 1. Poll the world
  BAL=$(poof data query -p $P --name getSolBalance --args "{\"address\":\"$SELF\"}" --json | jq -r .result)
  MEM=$(poof data get -p $P --path "memories/$SELF" --json | jq -r '.[0].content // "first run"')

  # 2. Decide
  if (( BAL < 100000000 )); then  # 100 USDC in 6-dec base units
    # 3. Act — guard + transfer in one atomic bundle
    cat > /tmp/topup.json <<EOF
    [
      { "path":"user/$SELF/TokenTransfer/topup-$(date +%s)",
        "document":{"source":"$VAULT","destination":"$SELF","mint":"$USDC","amount":50000000}},
      { "path":"user/$SELF/BalanceCheck/topup-$(date +%s)-floor",
        "document":{"mint":"$USDC","op":"gte","threshold":100000000}}
    ]
EOF
    poof data set-many -p $P --from-json /tmp/topup.json

    # 4. Record
    poof data set -p $P --path "memories/$SELF" \
      --data "{\"content\":\"topped up at $(date -u +%s), prev mem: $MEM\"}"
  fi

  sleep 60
done
```

Shape swaps — you'd read different queries, compose different guards, write to different state — but the skeleton stays the same.
