# Tokens

Create, transfer, and burn SPL tokens — both standard and Token-2022.

## TokenTransfer

Transfer SPL tokens between two Solana addresses. Amount is in the smallest unit (raw lamports/smallest-unit amounts with decimals applied).

**Collection:** `TokenTransfer/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Sender wallet (must match signer) |
| `destination` | `Address!` | Recipient wallet |
| `mint` | `Address!` | Token mint address |
| `amount` | `UInt!` | Amount in smallest unit (e.g., 1000000 for 1 USDC) |

**Rules:** Source wallet must match the transaction signer.

**Hook:** `@TokenPlugin.transfer(source, destination, mint, amount)`

**CLI Example:**
```bash
poof onchain set TokenTransfer/tx-001 \
  --data '{"source":"<your-wallet>","destination":"<recipient>","mint":"<token-mint>","amount":1000000}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## TokenTransferWholeTokens

Same as TokenTransfer but accepts human-readable whole token amounts. The plugin automatically applies the token's decimal conversion.

**Collection:** `TokenTransferWholeTokens/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Sender wallet (must match signer) |
| `destination` | `Address!` | Recipient wallet |
| `mint` | `Address!` | Token mint address |
| `amount` | `UInt!` | Amount in whole tokens (e.g., 100 = 100 USDC) |

**Rules:** Source wallet must match the transaction signer.

**Hook:** `@TokenPlugin.transferWholeTokens(source, destination, mint, amount)`

**CLI Example:**
```bash
poof onchain set TokenTransferWholeTokens/tx-001 \
  --data '{"source":"<your-wallet>","destination":"<recipient>","mint":"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v","amount":100}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## TokenCreate

Create a new standard SPL token with metadata.

**Collection:** `TokenCreate/$id`

| Field | Type | Description |
|-------|------|-------------|
| `tokenId` | `String!` | Unique ID used to derive the mint address |
| `name` | `String!` | Token name |
| `symbol` | `String!` | Token symbol |
| `uri` | `String!` | Metadata URI (image, attributes JSON) |
| `decimals` | `UInt!` | Decimal precision (e.g., 6 for USDC-like, 9 for SOL-like) |

**Rules:** Anyone can create a token. The `tokenId` must be unique.

**Hook:** `@TokenPlugin.createToken(tokenId, name, symbol, uri, decimals)`

**CLI Example:**
```bash
poof onchain set TokenCreate/my-token \
  --data '{"tokenId":"my-token-v1","name":"My Token","symbol":"MTK","uri":"https://arweave.net/...","decimals":6}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## TokenCreate2022

Create a Token-2022 (Token Extensions) token with advanced features. Only include the extension fields you need — omitted fields are ignored.

**Collection:** `TokenCreate2022/$id`

| Field | Type | Description |
|-------|------|-------------|
| `tokenId` | `String!` | Unique ID used to derive the mint address |
| `name` | `String!` | Token name |
| `symbol` | `String!` | Token symbol |
| `uri` | `String!` | Metadata URI |
| `decimals` | `UInt!` | Decimal precision |
| `nonTransferable` | `Bool?` | Make token non-transferable (soulbound) |
| `feeBasisPoints` | `UInt?` | Transfer fee in basis points (0-65535). Requires `maxFee` and `transferFeeAuthority` |
| `maxFee` | `UInt?` | Maximum fee per transfer |
| `transferFeeAuthority` | `Address?` | Who can modify fees |
| `withdrawWithheldAuthority` | `Address?` | Who can withdraw withheld fees (defaults to `transferFeeAuthority`) |
| `interestRate` | `Int?` | Interest rate (i16). Requires `interestRateAuthority` |
| `interestRateAuthority` | `Address?` | Who can modify interest rate |
| `permanentDelegate` | `Address?` | **DANGER:** Grants irrevocable unlimited transfer and burn authority over every holder's tokens. Cannot be undone. |

**Rules:** Anyone can create a Token-2022. Extension fields are all optional.

**Hook:** `@TokenPlugin.createToken2022(tokenId, name, symbol, uri, decimals, {extensions...})`

**CLI Example:**
```bash
poof onchain set TokenCreate2022/my-token-2022 \
  --data '{"tokenId":"my-2022-token","name":"Fee Token","symbol":"FEE","uri":"https://arweave.net/...","decimals":6,"feeBasisPoints":100,"maxFee":1000000,"transferFeeAuthority":"<your-wallet>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## TokenBurn

Permanently destroy SPL tokens, reducing total supply. Irreversible.

**Collection:** `TokenBurn/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Token holder wallet (must match signer) |
| `mint` | `Address!` | Token mint address |
| `amount` | `UInt!` | Amount to burn |

**Rules:** Source wallet must match the transaction signer.

**Hook:** `@TokenPlugin.burn(source, mint, amount)`

**CLI Example:**
```bash
poof onchain set TokenBurn/burn-001 \
  --data '{"source":"<your-wallet>","mint":"<token-mint>","amount":500000}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## TokenWithdrawWithheldTokens

Withdraw accumulated transfer fees from a Token-2022 mint with transfer fee extensions enabled.

**Collection:** `TokenWithdrawWithheldTokens/$id`

| Field | Type | Description |
|-------|------|-------------|
| `mint` | `Address!` | Token-2022 mint with transfer fees |
| `withdrawAuthority` | `Address!` | Withdraw authority (must match signer) |
| `feeReceiverOwner` | `Address!` | Where to send collected fees |
| `sourceOwner` | `Address!` | Token account to collect fees from |

**Rules:** Withdraw authority must match the transaction signer.

**Hook:** `@TokenPlugin.withdrawWithheldTokens(mint, withdrawAuthority, feeReceiverOwner, sourceOwner)`

**CLI Example:**
```bash
poof onchain set TokenWithdrawWithheldTokens/withdraw-001 \
  --data '{"mint":"<token-2022-mint>","withdrawAuthority":"<your-wallet>","feeReceiverOwner":"<your-wallet>","sourceOwner":"<source-wallet>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Related Queries

See [queries.md](queries.md) for read-only token data:

- `getTokenBalance` — Token balance for an address
- `getSolBalance` — SOL balance
- `getUsdcBalance` — USDC balance
- `getTokenDecimals` — Token decimal precision
- `getTokenSupply` — Total token supply
- `getTokenMintAddress` — Derived mint address for a token ID
- `getWithdrawWithheldAuthority` — Withdraw authority for a Token-2022 mint

**CLI Query Examples:**
```bash
poof onchain query queries/getTokenBalance getTokenBalance \
  --args '{"address":"<wallet>","mint":"<token-mint>"}' --app 69bcffc78d4b88997d0ed01a

poof onchain query queries/getSolBalance getSolBalance \
  --args '{"address":"<wallet>"}' --app 69bcffc78d4b88997d0ed01a

poof onchain query queries/getTokenMintAddress getTokenMintAddress \
  --args '{"tokenId":"my-token-v1"}' --app 69bcffc78d4b88997d0ed01a
```
