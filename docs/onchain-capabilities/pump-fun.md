# Pump.fun

Launch tokens on pump.fun with bonding curves, buy tokens, collect and distribute creator fees, and manage PumpSwap liquidity.

## Token Launches

### PumpLaunch

Launch a standard SPL token on pump.fun with an automatic bonding curve. The token starts trading immediately.

**Collection:** `PumpLaunch/$id`

| Field | Type | Description |
|-------|------|-------------|
| `tokenId` | `String!` | Unique ID for mint address derivation |
| `name` | `String!` | Token name |
| `symbol` | `String!` | Token symbol |
| `uri` | `String!` | Metadata URI |
| `creator` | `Address!` | Creator wallet (must match signer) |
| `seedMode` | `String?` | Set to `"idOnly"` for deterministic mint address from tokenId |

**Hook:** `@PumpFunPlugin.createToken(tokenId, name, symbol, uri, creator, {seedMode})`

**CLI Example:**
```bash
poof onchain set PumpLaunch/launch-001 \
  --data '{"tokenId":"my-token","name":"My Token","symbol":"MTK","uri":"https://arweave.net/...","creator":"<your-wallet>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

### PumpLaunchV2

Launch a Token-2022 token on pump.fun. Supports mayhem mode for experimental token mechanics.

**Collection:** `PumpLaunchV2/$id`

| Field | Type | Description |
|-------|------|-------------|
| `tokenId` | `String!` | Unique ID for mint address derivation |
| `name` | `String!` | Token name |
| `symbol` | `String!` | Token symbol |
| `uri` | `String!` | Metadata URI |
| `creator` | `Address!` | Creator wallet (must match signer) |
| `isMayhemMode` | `Bool!` | Enable experimental Token-2022 mechanics |
| `seedMode` | `String?` | Set to `"idOnly"` for deterministic mint address |

**Hook:** `@PumpFunPlugin.createTokenV2(tokenId, name, symbol, uri, creator, isMayhemMode, {seedMode})`

**CLI Example:**
```bash
poof onchain set PumpLaunchV2/launch-v2-001 \
  --data '{"tokenId":"my-token-v2","name":"My Token V2","symbol":"MTV2","uri":"https://arweave.net/...","creator":"<your-wallet>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Trading

### PumpBuy

Buy tokens on pump.fun by specifying exact SOL amount to spend. The bonding curve determines how many tokens you receive.

**Collection:** `PumpBuy/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Buyer wallet (must match signer) |
| `mint` | `Address!` | Token mint to buy |
| `solAmount` | `String!` | SOL amount in lamports |
| `slippageBps` | `String!` | Slippage tolerance in basis points (e.g., `"500"` = 5%) |

**Hook:** `@PumpFunPlugin.buyExactSolIn(source, mint, solAmount, slippageBps)`

**CLI Example:**
```bash
poof onchain set PumpBuy/buy-001 \
  --data '{"buyer":"<your-wallet>","tokenMint":"<token-mint>","solAmount":100000000}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Creator Fees

### PumpCollectCreatorFee

Collect accumulated creator trading fees from a pump.fun token.

**Collection:** `PumpCollectCreatorFee/$id`

| Field | Type | Description |
|-------|------|-------------|
| `creator` | `Address!` | Creator wallet (must match signer) |
| `mint` | `Address!` | Token mint |

**Hook:** `@PumpFunPlugin.collectCreatorFee(creator, mint)`

**CLI Example:**
```bash
poof onchain set PumpCollectCreatorFee/collect-001 \
  --data '{"creator":"<your-wallet>","tokenMint":"<token-mint>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

### PumpTransferCreatorFeesToPump

Transfer accumulated AMM trading fees from a graduated token back to the pump creator vault. Only needed for tokens that graduated from the bonding curve to a full AMM.

**Collection:** `PumpTransferCreatorFeesToPump/$id`

| Field | Type | Description |
|-------|------|-------------|
| `mint` | `Address!` | Token mint |

**Rules:** Permissionless — anyone can trigger this.

**Hook:** `@PumpFunPlugin.transferCreatorFeesToPump(mint)`

**CLI Example:**
```bash
poof onchain set PumpTransferCreatorFeesToPump/transfer-001 \
  --data '{"creator":"<your-wallet>","tokenMint":"<token-mint>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Fee Sharing

Fee sharing lets you split creator fees among multiple shareholders. Setup flow:

1. **Create config** → `PumpCreateFeeSharingConfig`
2. **Set shareholders** → `PumpUpdateShareholders`
3. **Distribute** → `PumpDistributeCreatorFees` (permissionless, anyone can trigger)

### PumpCreateFeeSharingConfig

Initialize the fee sharing configuration for a token. Must be called first before setting shareholders.

**Collection:** `PumpCreateFeeSharingConfig/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Initiator wallet (must match signer, usually the creator) |
| `mint` | `Address!` | Token mint |

**Hook:** `@PumpFunPlugin.createFeeSharingConfig(source, mint)`

**CLI Example:**
```bash
poof onchain set PumpCreateFeeSharingConfig/config-001 \
  --data '{"creator":"<your-wallet>","tokenMint":"<token-mint>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

### PumpUpdateShareholders

Set or replace all shareholders atomically. Total basis points must equal 10000 (100%).

**Collection:** `PumpUpdateShareholders/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Authority wallet (must match signer) |
| `mint` | `Address!` | Token mint |
| `shareholders` | `String!` | JSON array: `[{"addr": "...", "bps": 8000}, {"addr": "...", "bps": 2000}]` |

**Hook:** `@PumpFunPlugin.updateShareholders(source, mint, shareholders)`

**CLI Example:**
```bash
poof onchain set PumpUpdateShareholders/update-001 \
  --data '{"creator":"<your-wallet>","tokenMint":"<token-mint>","shareholders":["<addr1>","<addr2>"],"sharesBps":[5000,5000]}' \
  --app 69bcffc78d4b88997d0ed01a
```

### PumpDistributeCreatorFees

Distribute accumulated fees to all configured shareholders.

**Collection:** `PumpDistributeCreatorFees/$id`

| Field | Type | Description |
|-------|------|-------------|
| `mint` | `Address!` | Token mint |

**Rules:** Permissionless — anyone can trigger distribution.

**Hook:** `@PumpFunPlugin.distributeCreatorFees(mint)`

**CLI Example:**
```bash
poof onchain set PumpDistributeCreatorFees/distribute-001 \
  --data '{"tokenMint":"<token-mint>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## PumpSwap Liquidity

### PumpswapDeposit

Deposit liquidity into a PumpSwap AMM pool. Receive LP tokens representing your pool share.

**Collection:** `PumpswapDeposit/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Depositor wallet (must match signer) |
| `mint` | `Address!` | Pool token mint |
| `lpTokenAmountOut` | `String!` | Desired LP tokens to receive |
| `maxBaseAmountIn` | `String!` | Maximum base tokens to deposit |
| `maxQuoteAmountIn` | `String!` | Maximum quote tokens to deposit |

**Hook:** `@PumpFunPlugin.pumpswapDeposit(source, mint, lpTokenAmountOut, maxBaseAmountIn, maxQuoteAmountIn)`

**CLI Example:**
```bash
poof onchain set PumpswapDeposit/deposit-001 \
  --data '{"depositor":"<your-wallet>","tokenMint":"<token-mint>","tokenAmount":1000000,"solAmount":100000000}' \
  --app 69bcffc78d4b88997d0ed01a
```

### PumpswapWithdraw

Withdraw liquidity from a PumpSwap pool by burning LP tokens.

**Collection:** `PumpswapWithdraw/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Withdrawer wallet (must match signer) |
| `mint` | `Address!` | Pool token mint |
| `lpTokenAmountIn` | `String!` | LP tokens to burn |
| `minBaseAmountOut` | `String!` | Minimum base tokens to receive |
| `minQuoteAmountOut` | `String!` | Minimum quote tokens to receive |

**Hook:** `@PumpFunPlugin.pumpswapWithdraw(source, mint, lpTokenAmountIn, minBaseAmountOut, minQuoteAmountOut)`

**CLI Example:**
```bash
poof onchain set PumpswapWithdraw/withdraw-001 \
  --data '{"withdrawer":"<your-wallet>","tokenMint":"<token-mint>","lpTokenAmount":500000}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Related Queries

See [queries.md](queries.md):

- `getBondingCurveProgress` — How far along the bonding curve a token is
- `getPumpCreatorFee` — Accumulated creator fee for a token
