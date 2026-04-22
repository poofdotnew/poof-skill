# Onchain action catalog — generic-onchain primitives library

Every onchain action the generic-onchain Poof appid exposes as a collection you can write to via `poof data set` or `poof data set-many`. Writing a doc triggers the plugin hook that CPIs into the target Solana program; the collection's fields are exactly what the hook needs.

User-scoping (`user/$userId/<Collection>/$id` with `$userId == @user.address`) is enforced on every write. That means:

- A caller can only write under their own wallet prefix.
- Fees are paid by the caller's own wallet.
- Any caller with a Solana wallet can use the library — you don't need to own the Poof project.

For composition (guard + action in one atomic bundle), see [set-many.md](set-many.md). For the Phoenix.trade perps deep-dive, see [perps.md](perps.md). For the guard primitives used alongside these actions, see [guards.md](guards.md).

## Targeting

The canonical shared-appid for the generic-onchain primitives library on Solana mainnet is:

```
--app-id 69bcffc78d4b88997d0ed01a --chain mainnet
```

Anyone with a Solana wallet can write against it; user-scoping sandboxes callers to their own `user/<their-addr>/...` entries. Fees are paid by the caller's own wallet. This is the appid used in every example below.

```bash
# Shared-appid (preferred — no project access needed)
poof data set --app-id 69bcffc78d4b88997d0ed01a --chain mainnet \
  --path "user/<self>/<Collection>/<id>" --data '{...}'

# Or project-based, if you own a project with this policy deployed
poof data set -p <project-id> -e production \
  --path "user/<self>/<Collection>/<id>" --data '{...}'
```

If you've deployed the primitives library to your own project, `poof data app-ids -p <project-id>` lists its Tarobase appIds per environment so you can grab one for shared-appid mode.

## Token (SPL + SPL 2022)

| Action | Fields | Notes |
|---|---|---|
| `AccountCreate` | `accountId: String` | Creates an empty SPL account record. Rarely needed directly. |
| `TokenTransfer` | `source, destination, mint: Address`, `amount: UInt` | `amount` in base units (×10^decimals). |
| `TokenTransferWholeTokens` | same as `TokenTransfer` | `amount` in whole tokens; CLI auto-scales by mint decimals. |
| `TokenCreate` | `tokenId, name, symbol, uri: String`, `decimals: UInt` | SPL token mint. |
| `TokenCreate2022` | `tokenId, name, symbol, uri, decimals` + optional `nonTransferable, feeBasisPoints, maxFee, transferFeeAuthority, withdrawWithheldAuthority, interestRate, interestRateAuthority, permanentDelegate` | SPL 2022 with extensions. |
| `TokenBurn` | `source, mint: Address`, `amount: UInt` | Base units. |
| `TokenWithdrawWithheldTokens` | `mint, withdrawAuthority, feeReceiverOwner, sourceOwner: Address` | SPL 2022 transfer-fee withdraw. |

```bash
poof data set --app-id 69bcffc78d4b88997d0ed01a --chain mainnet \
  --path "user/<self>/TokenTransfer/tt-1" \
  --data '{"source":"<self>","destination":"<peer>","mint":"<usdc>","amount":50000000}'
```

## NFT (Metaplex Core + SPL NFTs)

| Action | Fields | Notes |
|---|---|---|
| `NftTransfer` | `source, destination, mint: Address`, `collectionAddress?: Address` | Metaplex Core + SPL NFTs. |
| `NftCreateCollection` | `collectionId, name, metadataUri: String` | Metaplex Core collection. |
| `NftBurn` | `source, mint: Address`, `collectionAddress?: Address` | — |

## Pump.fun

| Action | Fields | Notes |
|---|---|---|
| `PumpLaunch` | `tokenId, name, symbol, uri: String`, `creator: Address`, `seedMode?: String` | Legacy launch. |
| `PumpLaunchV2` | same as v1 + `isMayhemMode: Bool` | Current launch flow. |
| `PumpBuy` | `source, mint: Address`, `solAmount, slippageBps: String` | `solAmount` in lamports as string. |
| `PumpCollectCreatorFee` | `creator, mint: Address` | — |
| `PumpDistributeCreatorFees` | `mint: Address` | — |
| `PumpCreateFeeSharingConfig` | `source, mint: Address` | — |
| `PumpUpdateShareholders` | `source, mint: Address`, `shareholders: String` | JSON-array-as-string. |
| `PumpTransferCreatorFeesToPump` | `mint: Address` | — |
| `PumpswapDeposit` | `source, mint: Address`, `lpTokenAmountOut, maxBaseAmountIn, maxQuoteAmountIn: String` | LP deposit. |
| `PumpswapWithdraw` | `source, mint: Address`, `lpTokenAmountIn, minBaseAmountOut, minQuoteAmountOut: String` | LP withdraw. |

## Meteora (DBC + DAMM v2 CP-AMM)

| Action | Fields | Notes |
|---|---|---|
| `MeteoraCreatePool` | `source, tokenMintA, tokenMintB: Address`, `tokenAAmount, tokenBAmount: String` + many optional DBC/DAMM knobs | Full config surface; most optional fields can be omitted. |
| `MeteoraSwap` | `source, tokenMintA, tokenMintB: Address`, `tokenAAmount: String`, `slippageBps?: UInt` | A → B direction. |
| `MeteoraCreateConfig` | `configId, feeAccount`, `pre/postMigrated{FeeAmountBps,CreatorFeePercentage}` + optional market caps | DBC config. |
| `MeteoraCreateVirtualPool` | `configId, tokenId, name, symbol, uri`, `initialSolBuyAmount?` | DBC virtual pool. |
| `MeteoraSwapInVirtualPool` | `source, poolTokenMint, tokenMint, amount: String` | — |
| `MeteoraClaimPoolFees` | `source, poolAddress` | DBC pool fees. |
| `MeteoraClaimDammV2PoolFees` | `source, poolAddress, positionMintAddress?` | DAMM v2. |
| `MeteoraCreateCpAmmPosition` | `owner, poolAddress, positionId` | Opens a CP-AMM position. |
| `MeteoraAddCpAmmLiquidity` | `source, poolAddress, positionMintAddress`, `tokenAAmount, tokenBAmount`, `slippageBps?` | — |
| `MeteoraRemoveCpAmmLiquidity` | `source, poolAddress, positionMintAddress`, `tokenAAmount?, tokenBAmount?`, `slippageBps?` | Amounts optional → remove all. |
| `MeteoraLockCpAmmPosition` | `source, poolAddress, positionMintAddress`, `periodFrequency, cliffUnlockLiquidity, liquidityPerPeriod, numberOfPeriod`, `cliffPoint?` | Vesting schedule for LP. |
| `MeteoraCloseCpAmmPosition` | `source, poolAddress, positionMintAddress` | — |

## DFlow (prediction markets)

| Action | Fields | Notes |
|---|---|---|
| `DflowOpenPredictionMarketOrder` | `source, inputMint, outputMint`, `amount: UInt`, `slippageBps?` | Input → outcome-token swap. |

## Tensor (NFT marketplace)

| Action | Fields | Notes |
|---|---|---|
| `TensorBuyNft` | `assetAddress: Address`, `maxAmount: UInt` | **Known broken:** `assetAddress` is currently coerced to Number() server-side and overflows on base58 pubkeys. Plugin-side fix needed. |
| `TensorListNft` | `assetAddress: Address`, `amount?, expireInSec?: UInt`, `currency?, privateTaker?, makerBroker?` | Same Tensor-plugin overflow bug. |

## Phoenix.trade perps

| Action | Fields | Notes |
|---|---|---|
| `PhoenixRegister` | _(none)_ | One-shot trader PDA. |
| `PhoenixFund` | `amt: UInt` | USDC base units → Phoenix margin. |
| `PhoenixLong` | `market: Address`, `baseLots: UInt` | — |
| `PhoenixShort` | same as `PhoenixLong` | — |
| `PhoenixClose` | `market, baseLots`, `side: UInt` | `side: 1` closes long; `side: 0` closes short. Reduce-only. |

See [perps.md](perps.md) for the full Phoenix integration: canonical market addresses, the seven commonQueries reads (`isRegistered`, `portfolioValue`, `unrealizedPnl`, `positionSize`, `hasPosition`, `markPrice`, etc.), lots-to-dollars math, and composition patterns.

## Off-chain collections

| Collection | Fields | Notes |
|---|---|---|
| `memories/$userId` | `content: String` | Private per-user agent scratch. One doc per wallet; only the owner can read or write. Free (no chain fee). Good for agent state between ticks. |

```bash
# Off-chain collections against the shared mainnet appid still use
# --chain mainnet (the server routes offchain-only writes internally).
poof data set --app-id 69bcffc78d4b88997d0ed01a --chain mainnet \
  --path "memories/<self>" --data '{"content":"last run at <ts>, balance was ..."}'
```

## Read surface

- `queries/$queryId` — global reads (token balances, prices, pool metadata, bonding-curve math, etc.). Call with `poof data query --name <queryName>`.
- `commonQueries/$queryId` — per-domain dashboard reads. Phoenix uses this for its 7 reads. Call with `poof data query --path 'commonQueries/x' --name <queryName> --args '{...}'`.

Run `poof data query -p <project-id> --name <name>` without args to discover what's exposed; the server rejects unknown names with a clear error.

## When to reach for which

- **Immediate, unconditional action:** `poof data set --path "user/<self>/<Collection>/<id>" --data '...'`. Any rule failure rolls back the tx.
- **Gated action (action + precondition check):** `poof data set-many` with a guard collection ([guards.md](guards.md)) alongside the action. All-or-nothing atomic bundle.
- **Multi-leg action (swap then lock, deposit then fund, etc.):** same pattern — both writes in one `set-many` bundle.

See [set-many.md](set-many.md) for composition patterns and failure semantics.
