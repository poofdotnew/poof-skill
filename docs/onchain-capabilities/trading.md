# Trading

Prediction market orders and other trading operations.

## DflowOpenPredictionMarketOrder

Open a prediction market order through DFlow's escrowed swap mechanism. Escrows input tokens and creates an order filled when the market resolves.

**Collection:** `DflowOpenPredictionMarketOrder/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Trader wallet (must match signer) |
| `inputMint` | `Address!` | Input token mint |
| `outputMint` | `Address!` | Output token mint |
| `amount` | `UInt!` | Input amount |
| `slippageBps` | `UInt?` | Slippage tolerance in basis points |

**Hook:** `@DflowPlugin.openPredictionMarketOrder(source, inputMint, outputMint, amount, slippageBps)`

**CLI Example:**
```bash
poof onchain set DflowOpenPredictionMarketOrder/order-001 \
  --data '{"user":"<your-wallet>","marketAddress":"<market>","side":"buy","amount":1000000}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Related Queries

See [queries.md](queries.md):

- `getKycStatus` — Check DFlow KYC status for an address
- `getYesTokenOutAmm` / `getCollateralOutAmm` — AMM prediction market pricing
- `applyFee` — Apply fee to a prediction market amount
- `getYesTokensOutLsmr` / `getNoTokensOutLsmr` — LMSR prediction market pricing
- `getYesCollateralOutLsmr` / `getNoCollateralOutLsmr` — LMSR collateral calculations
