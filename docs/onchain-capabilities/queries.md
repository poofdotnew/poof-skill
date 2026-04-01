# Queries

Read-only, permissionless queries against the `queries/$queryId` collection. No transaction signing required. All queries are public.

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

## Account Queries

| Query | Returns | Description |
|-------|---------|-------------|
| `getAccountAddress` | `String` | PDA address for a given account ID |

## Price Feeds

Real-time prices via Pyth oracles.

| Query | Returns | Description |
|-------|---------|-------------|
| `getSolPriceInUSD` | `UInt` | Current SOL price in USD |
| `getBtcPriceInUSD` | `UInt` | Current BTC price in USD |
| `getEthPriceInUSD` | `UInt` | Current ETH price in USD |
| `getUsdcPriceInUSD` | `UInt` | Current USDC price in USD |
| `getPriceFeed` | `UInt` | Price for any supported Pyth feed |

## Pump.fun Queries

| Query | Returns | Description |
|-------|---------|-------------|
| `getBondingCurveProgress` | `UInt` | How far along the bonding curve a token is |
| `getPumpCreatorFee` | `UInt` | Accumulated creator fee for a token |

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

## NFT Queries

| Query | Returns | Description |
|-------|---------|-------------|
| `getNftBalance` | `UInt` | NFT balance for an address |
| `getNftMintAddress` | `String` | Derived NFT mint address |
| `getCollectionMintAddress` | `String` | Derived collection mint address |

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
