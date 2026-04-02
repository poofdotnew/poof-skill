# Meteora

Create liquidity pools, swap tokens, manage bonding curves, and operate CP-AMM positions on Meteora.

## Pools & Swaps

### MeteoraCreatePool

Create a new Meteora DLMM or dynamic liquidity pool. Supports extensive configuration — only include the fields you need, omitted fields use Meteora defaults.

**Collection:** `MeteoraCreatePool/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Creator wallet (must match signer) |
| `tokenMintA` | `Address!` | First token mint |
| `tokenMintB` | `Address!` | Second token mint |
| `tokenAAmount` | `String!` | Initial liquidity for token A |
| `tokenBAmount` | `String!` | Initial liquidity for token B |
| `baseFeeBps` | `UInt?` | Base trading fee in basis points |
| `numberOfPeriod` | `UInt?` | Total fee decay periods |
| `periodFrequency` | `UInt?` | Seconds between fee decay periods |
| `reductionFactor` | `UInt?` | Fee reduction per period |
| `feeSchedulerMode` | `String?` | Fee scheduler mode |
| `protocolFeePercent` | `UInt?` | Protocol fee percentage |
| `referralFeePercent` | `UInt?` | Referral fee percentage |
| `compoundingFeeBps` | `UInt?` | Compounding fee in basis points |
| `dynamicFeeEnabled` | `Bool?` | Auto-adjust fees based on volatility |
| `binStep` | `UInt?` | Tick spacing for concentrated liquidity |
| `filterPeriod` | `UInt?` | Dynamic fee filter period |
| `decayPeriod` | `UInt?` | Dynamic fee decay period |
| `dynamicFeeReductionFactor` | `UInt?` | Dynamic fee reduction factor |
| `maxVolatilityAccumulator` | `UInt?` | Max volatility accumulator |
| `variableFeeControl` | `UInt?` | Variable fee control |
| `collectFeeMode` | `String?` | Fee collection mode |
| `activationType` | `String?` | Activation type |
| `activationPoint` | `UInt?` | Activation point |
| `hasAlphaVault` | `Bool?` | Enable Alpha Vault for fair launch |

**Hook:** `@DeFiPlugin.createPool(source, tokenMintA, tokenMintB, tokenAAmount, tokenBAmount, {config...})`

**CLI Example:**
```bash
poof onchain set MeteoraCreatePool/pool-001 \
  --data '{"creator":"<your-wallet>","tokenMintA":"<mint-a>","tokenMintB":"<mint-b>","binStep":1,"activeId":0}' \
  --app 69bcffc78d4b88997d0ed01a
```

### MeteoraSwap

Swap tokens on Meteora. Default slippage is 500 bps (5%) if not specified.

**Collection:** `MeteoraSwap/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Swapper wallet (must match signer) |
| `tokenMintA` | `Address!` | Input token mint |
| `tokenMintB` | `Address!` | Output token mint |
| `tokenAAmount` | `String!` | Input amount |
| `slippageBps` | `UInt?` | Slippage tolerance (default: 500 = 5%) |

**Hook:** `@DeFiPlugin.swap(source, tokenMintA, tokenMintB, tokenAAmount, slippageBps)`

**CLI Example:**
```bash
poof onchain set MeteoraSwap/swap-001 \
  --data '{"user":"<your-wallet>","tokenMintIn":"<input-mint>","tokenMintOut":"<output-mint>","amountIn":1000000}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Bonding Curves (Virtual Pools)

Launch tokens with a bonding curve that graduates to a full DAMM v2 AMM. Setup flow:

1. **Create config** → `MeteoraCreateConfig` (defines fee structure, market caps, supply)
2. **Create virtual pool** → `MeteoraCreateVirtualPool` (launches the token on the curve)
3. **Trade** → `MeteoraSwapInVirtualPool` (buy/sell on the curve)
4. Token graduates to DAMM v2 automatically when the curve fills

### MeteoraCreateConfig

Create a bonding curve configuration template.

**Collection:** `MeteoraCreateConfig/$id`

| Field | Type | Description |
|-------|------|-------------|
| `configId` | `String!` | Unique config identifier |
| `feeAccount` | `Address!` | Fee recipient |
| `preMigratedFeeAmountBps` | `UInt!` | Fee bps before graduation |
| `preMigratedCreatorFeePercentage` | `UInt!` | Creator fee % before graduation (e.g., 50 = 50%) |
| `postMigratedFeeAmountBps` | `UInt!` | Fee bps after graduation |
| `postMigratedCreatorFeePercentage` | `UInt!` | Creator fee % after graduation |
| `initialMarketCap` | `UInt?` | Starting market cap |
| `migrationMarketCap` | `UInt?` | Market cap at which the curve graduates |
| `totalTokenSupply` | `UInt?` | Total token supply |
| `tokenBaseDecimal` | `UInt?` | Token decimal precision |

**Hook:** `@DeFiPlugin.createMeteoraConfig(configId, feeAccount, ...fees, initialMarketCap, migrationMarketCap, totalTokenSupply, tokenBaseDecimal)`

**CLI Example:**
```bash
poof onchain set MeteoraCreateConfig/config-001 \
  --data '{"creator":"<your-wallet>","feeBps":100,"migrationFeePercentage":10}' \
  --app 69bcffc78d4b88997d0ed01a
```

### MeteoraCreateVirtualPool

Create a virtual pool (bonding curve) for a new token. Token is tradeable immediately.

**Collection:** `MeteoraCreateVirtualPool/$id`

| Field | Type | Description |
|-------|------|-------------|
| `configId` | `String!` | References a MeteoraCreateConfig |
| `tokenId` | `String!` | Unique token ID |
| `name` | `String!` | Token name |
| `symbol` | `String!` | Token symbol |
| `uri` | `String!` | Metadata URI |
| `initialSolBuyAmount` | `String?` | Optional dev buy in lamports at launch |

**Rules:** Anyone can create a virtual pool.

**Hook:** `@DeFiPlugin.createMeteoraVirtualPool(configId, tokenId, name, symbol, uri, initialSolBuyAmount)`

**CLI Example:**
```bash
poof onchain set MeteoraCreateVirtualPool/vpool-001 \
  --data '{"creator":"<your-wallet>","configId":"<config-id>","tokenId":"my-launch-token","name":"My Token","symbol":"MTK","uri":"https://arweave.net/..."}' \
  --app 69bcffc78d4b88997d0ed01a
```

### MeteoraSwapInVirtualPool

Buy or sell tokens on a bonding curve before graduation.

**Collection:** `MeteoraSwapInVirtualPool/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Swapper wallet (must match signer) |
| `poolTokenMint` | `Address!` | Pool's token mint |
| `tokenMint` | `Address!` | Token to swap in |
| `amount` | `String!` | Swap amount |

**Hook:** `@DeFiPlugin.swapInMeteoraVirtualPool(source, poolTokenMint, tokenMint, amount)`

**CLI Example:**
```bash
poof onchain set MeteoraSwapInVirtualPool/vswap-001 \
  --data '{"user":"<your-wallet>","configAddress":"<config>","tokenMintAddress":"<token-mint>","tokenToSwapInMintAddress":"solana","tokenAmount":100000000}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Fee Claims

### MeteoraClaimPoolFees

Claim accumulated trading fees from a Meteora virtual pool.

**Collection:** `MeteoraClaimPoolFees/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Claimant wallet (must match signer) |
| `poolAddress` | `Address!` | Virtual pool address |

**Hook:** `@DeFiPlugin.claimMeteoraPoolFees(source, poolAddress)`

**CLI Example:**
```bash
poof onchain set MeteoraClaimPoolFees/claim-001 \
  --data '{"user":"<your-wallet>","configAddress":"<config>","tokenMintAddress":"<token-mint>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

### MeteoraClaimDammV2PoolFees

Claim trading fees from a graduated DAMM v2 pool.

**Collection:** `MeteoraClaimDammV2PoolFees/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Claimant wallet (must match signer) |
| `poolAddress` | `Address!` | DAMM v2 pool address |
| `positionMintAddress` | `Address?` | Specific position NFT (optional) |

**Hook:** `@DeFiPlugin.claimDammV2PoolFees(source, poolAddress, positionMintAddress)`

**CLI Example:**
```bash
poof onchain set MeteoraClaimDammV2PoolFees/claim-v2-001 \
  --data '{"user":"<your-wallet>","tokenMintAddress":"<token-mint>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## CP-AMM Positions

Manage concentrated liquidity positions in Meteora Constant Product AMM pools. Positions are represented as NFTs.

Flow: **Create position** → **Add liquidity** → **Earn fees** → **Remove liquidity** → **Close position**

### MeteoraCreateCpAmmPosition

Create a new liquidity position (mints a position NFT).

**Collection:** `MeteoraCreateCpAmmPosition/$id`

| Field | Type | Description |
|-------|------|-------------|
| `owner` | `Address!` | Position owner (must match signer) |
| `poolAddress` | `Address!` | CP-AMM pool address |
| `positionId` | `String!` | Unique ID to derive the position NFT mint |

**Hook:** `@DeFiPlugin.createCpAmmPosition(owner, poolAddress, positionId)`

**CLI Example:**
```bash
poof onchain set MeteoraCreateCpAmmPosition/pos-001 \
  --data '{"owner":"<your-wallet>","poolAddress":"<pool-address>","positionId":"my-position"}' \
  --app 69bcffc78d4b88997d0ed01a
```

### MeteoraAddCpAmmLiquidity

Add liquidity to an existing position.

**Collection:** `MeteoraAddCpAmmLiquidity/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Depositor wallet (must match signer) |
| `poolAddress` | `Address!` | CP-AMM pool address |
| `positionMintAddress` | `Address!` | Position NFT mint |
| `tokenAAmount` | `String!` | Token A deposit amount |
| `tokenBAmount` | `String!` | Token B deposit amount |
| `slippageBps` | `UInt?` | Price tolerance |

**Hook:** `@DeFiPlugin.addCpAmmLiquidity(source, poolAddress, positionMintAddress, tokenAAmount, tokenBAmount, slippageBps)`

**CLI Example:**
```bash
poof onchain set MeteoraAddCpAmmLiquidity/add-liq-001 \
  --data '{"owner":"<your-wallet>","poolAddress":"<pool-address>","positionId":"my-position","tokenAAmount":1000000,"tokenBAmount":1000000}' \
  --app 69bcffc78d4b88997d0ed01a
```

### MeteoraRemoveCpAmmLiquidity

Remove liquidity from a position. Specify amounts for one or both tokens.

**Collection:** `MeteoraRemoveCpAmmLiquidity/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Withdrawer wallet (must match signer) |
| `poolAddress` | `Address!` | CP-AMM pool address |
| `positionMintAddress` | `Address!` | Position NFT mint |
| `tokenAAmount` | `String?` | Token A withdrawal amount |
| `tokenBAmount` | `String?` | Token B withdrawal amount |
| `slippageBps` | `UInt?` | Price tolerance |

**Hook:** `@DeFiPlugin.removeCpAmmLiquidity(source, poolAddress, positionMintAddress, tokenAAmount, tokenBAmount, slippageBps)`

**CLI Example:**
```bash
poof onchain set MeteoraRemoveCpAmmLiquidity/rm-liq-001 \
  --data '{"owner":"<your-wallet>","poolAddress":"<pool-address>","positionId":"my-position","lpAmount":500000}' \
  --app 69bcffc78d4b88997d0ed01a
```

### MeteoraLockCpAmmPosition

Lock liquidity with a vesting schedule. Locked liquidity cannot be withdrawn until vesting completes.

**Collection:** `MeteoraLockCpAmmPosition/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Position owner (must match signer) |
| `poolAddress` | `Address!` | CP-AMM pool address |
| `positionMintAddress` | `Address!` | Position NFT mint |
| `periodFrequency` | `UInt!` | Seconds between releases |
| `cliffUnlockLiquidity` | `String!` | Amount released at cliff |
| `liquidityPerPeriod` | `String!` | Amount released per period |
| `numberOfPeriod` | `UInt!` | Total vesting periods |
| `cliffPoint` | `UInt?` | Cliff timestamp |

**Hook:** `@DeFiPlugin.lockCpAmmPosition(source, poolAddress, positionMintAddress, periodFrequency, cliffUnlockLiquidity, liquidityPerPeriod, numberOfPeriod, cliffPoint)`

**CLI Example:**
```bash
poof onchain set MeteoraLockCpAmmPosition/lock-001 \
  --data '{"owner":"<your-wallet>","poolAddress":"<pool-address>","positionId":"my-position","vestingDuration":86400}' \
  --app 69bcffc78d4b88997d0ed01a
```

### MeteoraCloseCpAmmPosition

Close an empty position and reclaim rent. Position must have zero remaining liquidity.

**Collection:** `MeteoraCloseCpAmmPosition/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Position owner (must match signer) |
| `poolAddress` | `Address!` | CP-AMM pool address |
| `positionMintAddress` | `Address!` | Position NFT mint |

**Hook:** `@DeFiPlugin.closeCpAmmPosition(source, poolAddress, positionMintAddress)`

**CLI Example:**
```bash
poof onchain set MeteoraCloseCpAmmPosition/close-001 \
  --data '{"owner":"<your-wallet>","poolAddress":"<pool-address>","positionId":"my-position"}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Related Queries

See [queries.md](queries.md):

- `getMeteoraVirtualPoolAddress` — Virtual pool address for a config + token pair
- `getClaimableMeteoraPoolFees` — Unclaimed fees from a virtual pool
- `getDammV2PoolAddress` — Graduated DAMM v2 pool address
- `getSwapQuote` — Swap quote from Jupiter/DeFi aggregator
- `getMeteoraSwapQuote` — Swap quote specifically from Meteora pools
- `getCpAmmPositionNftMintAddress` — Position NFT mint address
- `getCpAmmPoolAddress` — CP-AMM pool address for a token pair
- `getClaimableCpAmmPositionFee` — Unclaimed fees from a CP-AMM position
