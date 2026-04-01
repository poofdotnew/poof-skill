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
