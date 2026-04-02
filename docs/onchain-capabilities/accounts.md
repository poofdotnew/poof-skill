# Accounts

Create on-chain PDA (Program Derived Address) accounts. Each account is derived from a unique ID and can hold SOL, tokens, and NFTs. Used as the foundation for custodial wallets, vaults, escrows, and any on-chain identity that needs its own address.

## AccountCreate

**Collection:** `AccountCreate/$id`

| Field | Type | Description |
|-------|------|-------------|
| `accountId` | `String!` | Unique identifier used to derive the PDA address |

**Rules:**
- Read: Public
- Create: Anyone can create an account PDA

**Hook:** `@AccountPlugin.createAccount(accountId)`

The `accountId` determines the derived address deterministically — the same ID always produces the same PDA.

**CLI Example:**
```bash
poof onchain set AccountCreate/vault-001 \
  --data '{"accountId":"my-vault"}' \
  --app 69bcffc78d4b88997d0ed01a
```

Query the derived PDA address:
```bash
poof onchain query queries/getAccountAddress getAccountAddress \
  --args '{"accountId":"my-vault"}' --app 69bcffc78d4b88997d0ed01a
```

### Example

```json
{
  "accountId": "vault-001"
}
```

### Related Queries

- `getAccountAddress` — Get the PDA address for a given account ID (see [queries.md](queries.md))
