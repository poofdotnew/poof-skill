# NFTs

Create collections, transfer, and burn NFTs on Solana. Buy and list NFTs on Tensor marketplace.

## Core Operations

### NftCreateCollection

Create a new NFT collection. Individual NFTs can be verified against this collection. The `collectionId` derives the collection mint address deterministically.

**Collection:** `NftCreateCollection/$id`

| Field | Type | Description |
|-------|------|-------------|
| `collectionId` | `String!` | Unique ID to derive the collection mint address |
| `name` | `String!` | Collection name |
| `metadataUri` | `String!` | URI pointing to JSON with image and attributes |

**Rules:** Anyone can create a collection.

**Hook:** `@NFTPlugin.createCollection(collectionId, name, metadataUri)`

**CLI Example:**
```bash
poof onchain set NftCreateCollection/my-collection \
  --data '{"collectionId":"my-nft-collection","name":"My NFTs","symbol":"MNFT","uri":"https://arweave.net/..."}' \
  --app 69bcffc78d4b88997d0ed01a
```

### NftTransfer

Transfer a compressed or uncompressed NFT. Handles both Metaplex and Token-2022 standards.

**Collection:** `NftTransfer/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | Sender wallet (must match signer) |
| `destination` | `Address!` | Recipient wallet |
| `mint` | `Address!` | NFT mint address |
| `collectionAddress` | `Address?` | Collection address (recommended for collection-verified NFTs) |

**Hook:** `@NFTPlugin.transfer(source, destination, mint, collectionAddress)`

**CLI Example:**
```bash
poof onchain set NftTransfer/transfer-001 \
  --data '{"owner":"<your-wallet>","destination":"<recipient>","mint":"<nft-mint-address>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

### NftBurn

Permanently burn an NFT. Irrecoverable. Reclaims rent from token and metadata accounts.

**Collection:** `NftBurn/$id`

| Field | Type | Description |
|-------|------|-------------|
| `source` | `Address!` | NFT owner wallet (must match signer) |
| `mint` | `Address!` | NFT mint address |
| `collectionAddress` | `Address?` | Collection address (may be required for collection-verified NFTs) |

**Hook:** `@NFTPlugin.burn(source, mint, collectionAddress)`

**CLI Example:**
```bash
poof onchain set NftBurn/burn-001 \
  --data '{"owner":"<your-wallet>","mint":"<nft-mint-address>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Tensor Marketplace

### TensorBuyNft

Buy an NFT listed on Tensor. The transaction fails if the listing price exceeds your `maxAmount`.

**Collection:** `TensorBuyNft/$id`

| Field | Type | Description |
|-------|------|-------------|
| `assetAddress` | `Address!` | NFT mint address |
| `maxAmount` | `UInt!` | Maximum price in lamports |

**Rules:** Anyone can buy.

**Hook:** `@TensorPlugin.buyNft(assetAddress, maxAmount)`

**CLI Example:**
```bash
poof onchain set TensorBuyNft/buy-001 \
  --data '{"buyer":"<your-wallet>","mint":"<nft-mint>","maxPrice":1000000000,"owner":"<seller-wallet>"}' \
  --app 69bcffc78d4b88997d0ed01a
```

### TensorListNft

List an NFT for sale on Tensor. The NFT remains in your wallet until sold.

**Collection:** `TensorListNft/$id`

| Field | Type | Description |
|-------|------|-------------|
| `assetAddress` | `Address!` | NFT mint address |
| `amount` | `UInt?` | Price in lamports |
| `expireInSec` | `UInt?` | Listing duration in seconds |
| `currency` | `String?` | Currency (defaults to SOL) |
| `privateTaker` | `Address?` | Restrict purchase to a specific buyer (OTC deals) |
| `makerBroker` | `Address?` | Broker for fee routing |

**Rules:** Signer must own the NFT.

**Hook:** `@TensorPlugin.listNft(assetAddress, amount, expireInSec, currency, privateTaker, makerBroker)`

**CLI Example:**
```bash
poof onchain set TensorListNft/list-001 \
  --data '{"owner":"<your-wallet>","mint":"<nft-mint>","price":2000000000}' \
  --app 69bcffc78d4b88997d0ed01a
```

---

## Related Queries

See [queries.md](queries.md):

- `getNftBalance` — NFT balance for an address
- `getNftMintAddress` — Derived NFT mint address
- `getCollectionMintAddress` — Derived collection mint address
