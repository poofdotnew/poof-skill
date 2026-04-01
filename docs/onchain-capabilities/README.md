# On-Chain Capabilities

Poof provides a central policy with pre-built collections for common Solana on-chain operations. Instead of building custom policies for token transfers, swaps, NFT minting, etc., agents can execute these operations directly against the central app.

## Central App ID

```
69bcffc78d4b88997d0ed01a
```

All operations use this project ID via the `-p` flag:

```bash
poof files get -p 69bcffc78d4b88997d0ed01a
```

This is separate from your own project — you can use both in the same session by passing `-p` explicitly.

## How It Works

Every collection in this policy is **on-chain** and **passthrough** (`isPassthrough: true`), meaning:

- **No data is stored** — the collection triggers an on-chain transaction and that's it
- **Public reads** — all collections have `"read": "true"`
- **Signer validation** — write operations require the transaction signer to match the relevant address field (e.g., `source`, `creator`, `owner`)

To execute an operation, write a document to the appropriate collection path. The on-chain hook fires automatically.

## Capability Groups

| Doc | What it covers |
|-----|----------------|
| [**Accounts**](accounts.md) | Create PDA accounts for vaults, escrows, custodial wallets |
| [**Tokens**](tokens.md) | Transfer, create (SPL & Token-2022), burn, and manage SPL tokens |
| [**Pump.fun**](pump-fun.md) | Token launches, buys, fee collection, fee sharing, PumpSwap LP |
| [**Meteora**](meteora.md) | Pool creation, swaps, bonding curves, CP-AMM positions, liquidity locking |
| [**NFTs**](nfts.md) | Create collections, transfer, burn NFTs; buy/list on Tensor |
| [**Trading**](trading.md) | DFlow prediction market orders |
| [**Queries**](queries.md) | Read-only data — balances, prices, pool addresses, swap quotes, and more |

## Quick Examples

```bash
# Check SOL balance of an address
# (uses the queries collection — see queries.md)

# Transfer 100 USDC (whole tokens) from your wallet
# Write to TokenTransferWholeTokens/<unique-id> with:
#   source: <your-wallet>
#   destination: <recipient>
#   mint: EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
#   amount: 100

# Launch a token on pump.fun
# Write to PumpLaunch/<unique-id> with:
#   tokenId: "my-token"
#   name: "My Token"
#   symbol: "MTK"
#   uri: "<metadata-uri>"
#   creator: <your-wallet>
```

## Field Type Reference

| Type | Description |
|------|-------------|
| `String!` | Required string |
| `String?` | Optional string |
| `Address!` | Required Solana public key (base58) |
| `Address?` | Optional Solana public key |
| `UInt!` | Required unsigned integer |
| `UInt?` | Optional unsigned integer |
| `Int?` | Optional signed integer |
| `Bool!` | Required boolean |
| `Bool?` | Optional boolean |
