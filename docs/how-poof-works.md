# How Poof Works

This is the most important doc for an agent orchestrating Poof. Understanding how Poof works means you can write better prompts to the `chat` tool — you'll know what's possible, what to ask for, and how to frame requests so the Poof AI builds the right thing.

## Contents
- [Core Concept](#core-concept)
- [Architecture Tiers](#architecture-tiers)
- [The Policy System](#the-policy-system)
- [On-Chain vs Off-Chain](#on-chain-vs-off-chain)
- [Plugin Ecosystem](#plugin-ecosystem)
- [Generated Database SDK](#generated-database-sdk)
- [What Poof Can and Can't Do](#what-poof-can-and-cant-do)
- [Why This Matters for Your Agent](#why-this-matters-for-your-agent)
- [Technology Translation](#technology-translation)

## Core Concept

Poof is a **fully-managed Backend-as-a-Service** for Solana. You don't write backend code for most features. Instead, you describe your data model, access rules, and blockchain operations in a **policy** (JSON), and Poof handles everything — storage, auth, real-time sync, and on-chain execution.

**The key insight:** 80% of apps need only **frontend + policy**. No custom backend. The Poof AI generates both when you chat with it.

## Architecture Tiers

| Tier | Components | When to use | Example |
|------|-----------|-------------|---------|
| 1 | Frontend only | Prototyping, mock data | Static landing page |
| 2 | Frontend + Policy | **Most apps (80%)** | Social app, marketplace, dashboard, game |
| 3 | Frontend + Policy + Backend | External API secrets, server-signed txns | OpenAI integration, Stripe payments |

When chatting with Poof, **don't ask for a backend unless you actually need external API secrets**. The policy system handles data, auth, rules, and blockchain.

## The Policy System

A policy is a JSON file (`poof.json`) that defines:

### Collections (Data Schema)
Firestore-like paths with dynamic segments:
- `users/$userId` — user profiles
- `posts/$postId` — blog posts
- `users/$userId/stakes/$tokenMint` — nested sub-collections

**Rules:** All path segments after the collection name must be dynamic (`$` prefix). For example, `users/$userId` is valid but `users/alice` is not. A path like `config/app` means the collection is `config` and `app` is a specific document ID — this is valid for single-document patterns (e.g., app settings), but the policy schema must define it as `config/$configId`.

### Fields (5 Types Only)
`String`, `Address`, `UInt`, `Int`, `Bool`

Modifiers: `?` (optional), `!` (readonly)

**No arrays or objects.** Model lists as sub-collections (e.g., `org/$orgId/members/$memberId` instead of `members: Address[]`).

**No Timestamp type.** Use `UInt` with Unix seconds.

### Rules (Access Control)
JavaScript-like expressions controlling read/create/update/delete:

```
"read": "true"                                    — public
"create": "@newData.owner == @user.address"        — owner creates
"update": "@data.owner == @user.address"           — owner updates
"delete": "@user.address == @constants.ADMIN"      — admin deletes
```

**Default deny** — omitted operations default to `false`.

Key variables:
- `@user.address` — current wallet
- `@data` / `@newData` — document before/after mutation
- `@constants.*` — app constants (ADMIN_ADDRESS, SOL, USDC, PROJECT_VAULT_ADDRESS)
- `@time.now` — server timestamp (seconds)
- `get(/path/$id)` — read another document
- `getAfter(/path/$id)` — read document after pending atomic changes

### Hooks (Blockchain Operations)
Policy hooks trigger on-chain actions automatically:

```json
"hooks": {
  "onchain": {
    "create": "@TokenPlugin.transfer(@user.address, @constants.PROJECT_VAULT_ADDRESS, @constants.USDC, @newData.amount)"
  }
}
```

This means: when a document is created, transfer USDC from the user to the vault. No backend code needed.

### Queries (Read-Only Helpers)
Policy `queries` handle read operations like checking balances, getting token addresses, or fetching swap quotes — without needing a backend endpoint.

## On-Chain vs Off-Chain

This is the most important architectural decision on Poof.

| | On-chain (`"onchain": true`) | Off-chain (`"onchain": false`) |
|---|---|---|
| **Use for** | Token transfers, staking, escrow, payments, ownership, VRF | Profiles, posts, comments, metadata, analytics, chat |
| **Reads** | Must be `"read": "true"` (public on blockchain) | Can have restricted reads |
| **Cost** | ~0.002-0.01 SOL rent per record | Free |
| **Speed** | Slower (blockchain confirmation) | Fast |
| **Cross-access** | Cannot read off-chain data | Can read on-chain data |
| **Fields** | Keep to 4-6 max (Solana tx size limits), use short names | No limit |

**Default off-chain.** Only go on-chain when you need blockchain guarantees (finance, ownership, verifiable randomness).

### Passthrough Pattern
For one-time operations (tips, direct transfers) where you don't need to store a record:

```json
{
  "onchain": true,
  "isPassthrough": true,
  "hooks": { "onchain": { "create": "@TokenPlugin.transfer(...)" } }
}
```

No rent cost — executes the hook but doesn't create persistent state.

### Rent Reclaim Pattern
Users pay rent on create → reclaim it on delete (plus any winnings/refunds via delete hooks).

## Plugin Ecosystem

Poof abstracts all blockchain operations through plugins. **Never import `@solana/web3.js` or `@solana/spl-token` directly.**

| Plugin | What it does |
|--------|-------------|
| `@TokenPlugin` | Create, transfer, mint, burn tokens. Includes Token-2022 (transfer fees, soulbound, seizure). |
| `@DeFiPlugin` | Swaps (Jupiter), liquidity pools, **Meteora bonding curves** (default for token launches), CP-AMM. |
| `@PumpFunPlugin` | Pump.fun-style token launches with auto bonding curve. |
| `@NFTPlugin` | Create collections, mint/transfer/burn NFTs. |
| `@TensorPlugin` | Buy/list NFTs on Tensor marketplace. |
| `@AccountPlugin` | Create PDAs for escrow/vaults. |
| `@OraclePlugin` | Verifiable randomness (VRF) for games, lotteries, gambling. |
| `@DocumentPlugin` | Cross-document atomic updates. Offchain hooks for chart/time-series data. |
| `@PriceFeedPlugin` | Real-time SOL/BTC/ETH price feeds. |
| `@PredictionMarketPlugin` | LMSR/AMM pricing for custom prediction markets. |
| `@DflowPlugin` | Tokenized Kalshi prediction markets. |
| `@BondingCurvePlugin` | Advanced custom bonding curve math. |
| `@MathPlugin` | Overflow-safe multiplication/division for large numbers. |

### Token Launches
- **Tradeable (default):** `@DeFiPlugin.createMeteoraVirtualPool` — automatic bonding curve, zero liquidity setup
- **Pump.fun style:** `@PumpFunPlugin.createToken` — only when user specifically wants Pump.fun
- **Non-tradeable:** `@TokenPlugin.createToken` — for airdrops, fixed sales, no trading

### Token Decimals
- SOL: 9 decimals
- USDC: 6 decimals
- Most tokens: 6 decimals (default)

## Generated Database SDK

When the Poof AI builds a project, it generates a **typed database SDK** from the policy. This is how all code (frontend and backend) interacts with Poof Cloud data.

**Structure:** `db-client.ts` (shared utilities) + `collections/<name>.ts` (per-collection typed functions)

| Location | Context | SDK under the hood |
|----------|---------|-------------------|
| `src/lib/db-client.ts` + `src/lib/collections/` | Frontend (React) | `@pooflabs/web` |
| `partyserver/src/db-client.ts` + `partyserver/src/collections/` | Backend (Hono) | `@pooflabs/server` |

Each collection file exports typed functions: `setItems()`, `getItems()`, `getManyItems()`, `subscribeItems()`, `subscribeManyItems()`, `deleteItems()`, `buildItems()` (for batch `setMany`), plus `ItemsRequest` and `ItemsResponse` types. Names are hierarchical — nested `users/$userId/stakes/$tokenMint` produces `setUsersStakes()`, `UsersStakesRequest`, etc.

**These files are generated — do not modify them directly.** After policy changes, the Poof AI regenerates them automatically. See [database-sdk.md](database-sdk.md) for the full reference.

## What Poof Can and Can't Do

### Can Do
- Data persistence (on-chain + off-chain)
- Real-time subscriptions (WebSocket)
- Wallet authentication
- All Solana token/NFT/DeFi operations (via plugins)
- Verifiable randomness (VRF)
- Prediction markets
- Access control and business logic (policy rules)
- Cross-document atomic operations

### Cannot Do
- ML/AI model training → use external API via backend
- Video/audio processing → use external service via backend
- Non-Solana blockchains → Solana only
- Native SOL staking → use liquid staking tokens via `@DeFiPlugin.swap`

### Key Constraints
- On-chain collections must have `"read": "true"`
- On-chain rules cannot access off-chain data (but off-chain can access on-chain)
- No array/object fields — use sub-collections
- No built-in role system — compare `@user.address` against admin constants
- Path segments must alternate: `collection/$id/collection/$id` (no two dynamic segments in a row)

## Why This Matters for Your Agent

When you send a `chat` message to the Poof AI, you're essentially telling it what to build. Knowing the above means you can:

1. **Ask for the right things** — "Add a staking vault using @AccountPlugin" instead of vague "add staking"
2. **Avoid impossible requests** — don't ask for Ethereum integration or array fields
3. **Make smart architecture calls** — tell it to keep profiles off-chain and payments on-chain
4. **Use the right patterns** — passthrough for tips, VRF for randomness, sub-collections for lists
5. **Skip the backend** — most features don't need one; only add backend when you need external API secrets

## Technology Translation

If you're thinking in traditional web dev terms:

| You're thinking... | On Poof, use... |
|---|---|
| PostgreSQL / MongoDB | Policy collections (off-chain) |
| Prisma / Drizzle / ORM | Generated `db-client` + `collections/` SDK |
| REST API | Policy rules + queries |
| WebSocket server | Poof Cloud subscriptions (automatic) |
| JWT / OAuth | Wallet-based auth |
| Ethereum / Polygon | Solana (via plugins) |
| AWS Lambda | Backend hooks (only if needed) |
| Redis / cache | Poof Cloud subscriptions (real-time) |
