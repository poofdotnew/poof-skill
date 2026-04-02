# Queries

Read-only, permissionless queries against the `queries/$queryId` collection. No transaction signing required. All queries are public.

## CLI Usage

All queries use the same CLI pattern:

```bash
poof onchain query queries/<queryName> <queryName> --args '{...}' --app 69bcffc78d4b88997d0ed01a
```

The first argument is the query path (`queries/<queryName>`), the second is the query name, and `--args` passes the query parameters as JSON.

## Token & Balance Queries

| Query | Returns | Description |
|-------|---------|-------------|
| `getTokenBalance` | `UInt` | Token balance for an address + mint |
| `getSolBalance` | `UInt` | SOL balance for an address |
| `getUsdcBalance` | `UInt` | USDC balance for an address |
| `getTokenDecimals` | `UInt` | Decimal precision for a token mint |
| `getTokenSupply` | `UInt` | Total supply of a token |
| `getTokenMintAddress` | `String` | Derived mint address for a token ID |
| `getWithdrawWithheldAuthority` | `String` | Withdraw authority for a Token-2022 mint |

**CLI Examples:**
```bash
poof onchain query queries/getSolBalance getSolBalance \
  --args '{"address":"<wallet>"}' --app 69bcffc78d4b88997d0ed01a

poof onchain query queries/getTokenBalance getTokenBalance \
  --args '{"address":"<wallet>","mint":"<token-mint>"}' --app 69bcffc78d4b88997d0ed01a

poof onchain query queries/getTokenMintAddress getTokenMintAddress \
  --args '{"tokenId":"my-token"}' --app 69bcffc78d4b88997d0ed01a
```

## Account Queries

| Query | Returns | Description |
|-------|---------|-------------|
| `getAccountAddress` | `String` | PDA address for a given account ID |

**CLI Example:**
```bash
poof onchain query queries/getAccountAddress getAccountAddress \
  --args '{"accountId":"my-vault"}' --app 69bcffc78d4b88997d0ed01a
```

## Price Feeds

Real-time prices via Pyth oracles.

| Query | Returns | Description |
|-------|---------|-------------|
| `getSolPriceInUSD` | `UInt` | Current SOL price in USD |
| `getBtcPriceInUSD` | `UInt` | Current BTC price in USD |
| `getEthPriceInUSD` | `UInt` | Current ETH price in USD |
| `getUsdcPriceInUSD` | `UInt` | Current USDC price in USD |
| `getPriceFeed` | `UInt` | Price for any supported Pyth feed |

**CLI Examples:**
```bash
poof onchain query queries/getSolPriceInUSD getSolPriceInUSD \
  --app 69bcffc78d4b88997d0ed01a

poof onchain query queries/getPriceFeed getPriceFeed \
  --args '{"feedId":"<pyth-feed-id>"}' --app 69bcffc78d4b88997d0ed01a
```

## Pump.fun Queries

| Query | Returns | Description |
|-------|---------|-------------|
| `getBondingCurveProgress` | `UInt` | How far along the bonding curve a token is |
| `getPumpCreatorFee` | `UInt` | Accumulated creator fee for a token |

**CLI Examples:**
```bash
poof onchain query queries/getBondingCurveProgress getBondingCurveProgress \
  --args '{"tokenMint":"<token-mint>"}' --app 69bcffc78d4b88997d0ed01a

poof onchain query queries/getPumpCreatorFee getPumpCreatorFee \
  --args '{"tokenMint":"<token-mint>"}' --app 69bcffc78d4b88997d0ed01a
```

## Meteora & DeFi Queries

| Query | Returns | Description |
|-------|---------|-------------|
| `getMeteoraVirtualPoolAddress` | `String` | Virtual pool address for a config + token pair |
| `getClaimableMeteoraPoolFees` | `UInt` | Unclaimed fees from a virtual pool |
| `getDammV2PoolAddress` | `String` | Graduated DAMM v2 pool address |
| `getSwapQuote` | `String` | Swap quote from Jupiter/DeFi aggregator |
| `getMeteoraSwapQuote` | `String` | Swap quote specifically from Meteora pools |
| `getCpAmmPositionNftMintAddress` | `String` | Position NFT mint for a CP-AMM position |
| `getCpAmmPoolAddress` | `String` | CP-AMM pool address for a token pair |
| `getClaimableCpAmmPositionFee` | `UInt` | Unclaimed fees from a CP-AMM position |

**CLI Examples:**
```bash
poof onchain query queries/getSwapQuote getSwapQuote \
  --args '{"inputMint":"solana","outputMint":"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v","amount":"1000000000"}' \
  --app 69bcffc78d4b88997d0ed01a

poof onchain query queries/getMeteoraSwapQuote getMeteoraSwapQuote \
  --args '{"tokenMintAddress":"<token-mint>","tokenToSwapInMintAddress":"solana","tokenAmount":"100000000"}' \
  --app 69bcffc78d4b88997d0ed01a
```

## NFT Queries

| Query | Returns | Description |
|-------|---------|-------------|
| `getNftBalance` | `UInt` | NFT balance for an address |
| `getNftMintAddress` | `String` | Derived NFT mint address |
| `getCollectionMintAddress` | `String` | Derived collection mint address |

**CLI Example:**
```bash
poof onchain query queries/getNftBalance getNftBalance \
  --args '{"address":"<wallet>","collectionId":"my-collection"}' --app 69bcffc78d4b88997d0ed01a
```

## Oracle & VRF

| Query | Returns | Description |
|-------|---------|-------------|
| `getRandomNumber` | `UInt` | Verifiable random number |
| `getVRFAddress` | `String` | VRF account address |

## DFlow

| Query | Returns | Description |
|-------|---------|-------------|
| `getKycStatus` | `Bool` | KYC verification status for an address |

## Bonding Curve Math

Calculations for custom bonding curve implementations.

| Query | Returns | Description |
|-------|---------|-------------|
| `getSolOutProduct` | `UInt` | SOL out for a product bonding curve sell |
| `getTokensOutProduct` | `UInt` | Tokens out for a product bonding curve buy |
| `getMaxSolInProduct` | `UInt` | Maximum SOL input for a product curve |
| `getMaxTokensInProduct` | `UInt` | Maximum tokens input for a product curve |
| `getMarketCapInSol` | `UInt` | Market cap in SOL for a bonding curve |

## Prediction Market Math

| Query | Returns | Description |
|-------|---------|-------------|
| `getYesTokenOutAmm` | `UInt` | Yes tokens out from AMM |
| `getCollateralOutAmm` | `UInt` | Collateral out from AMM |
| `applyFee` | `UInt` | Amount after fee applied |
| `getYesTokensOutLsmr` | `UInt` | Yes tokens out from LMSR |
| `getNoTokensOutLsmr` | `UInt` | No tokens out from LMSR |
| `getYesCollateralOutLsmr` | `UInt` | Yes collateral out from LMSR |
| `getNoCollateralOutLsmr` | `UInt` | No collateral out from LMSR |

## Math Utilities

| Query | Returns | Description |
|-------|---------|-------------|
| `getRandom` | `UInt` | Pseudo-random number in a range |
| `mulDivFloor` | `UInt` | Overflow-safe `(a * b) / c` rounded down |
| `mulDivCeil` | `UInt` | Overflow-safe `(a * b) / c` rounded up |

## String Utilities

| Query | Returns | Description |
|-------|---------|-------------|
| `stringLength` | `UInt` | Length of a string |
