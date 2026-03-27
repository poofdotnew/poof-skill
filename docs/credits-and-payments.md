# Credits & Payments

## Contents
- [Credit System](#credit-system)
- [Paid Features](#paid-features)
- [x402 Credit Top-Up](#x402-credit-top-up)
- [AI Preferences](#ai-preferences)

## Important: How Payment Unlocks Features

Poof gates paid features (mainnet deployment, code downloads, custom domains) on whether the user has **ever completed any credit purchase**. There is no separate membership or subscription required.

| What you need | How to get it |
|---------------|---------------|
| **AI credits** (for `chat`, `create_project`, builds) | Free daily allotment (~10/day), OR buy add-on credits via x402 USDC |
| **Paid features** (mainnet deploy, code downloads, custom domains) | Complete any credit purchase. Once you've purchased credits, paid features are permanently unlocked for that wallet |

**Key point:** A single x402 credit purchase unlocks both AI credits AND paid features (deployment, downloads, etc.). The `check_publish_eligibility` check passes as long as `hasUserEverPaid` is true for the wallet.

## Credit System

Poof uses credits for AI interactions (chat messages, builds). Check your balance before starting long workflows:

```typescript
const credits = await mcpCall('tools/call', {
  name: 'get_credits',
  arguments: {},
});
// Returns: daily (free) and add-on credits with totals
```

### Credit Types

| Type | Source | Details |
|------|--------|---------|
| Daily | Free | ~10 credits, reset daily, available to all users |
| Add-on | x402 purchase | Bought with USDC, expire in 6 months |

### Credit Cost Estimates

| Operation | Approx. Credits |
|-----------|----------------|
| `create_project` (full build) | 1-2 |
| `chat` (follow-up message) | 1 |
| Full build + test + polish cycle | 3-5 |

### Handling Credit Exhaustion

Always check credits before starting and between steps. If credits run out mid-build, the AI stops responding — `chat` calls will return errors and `check_ai_active` may hang.

```typescript
async function ensureCredits(minRequired: number) {
  const credits = await mcpCall('tools/call', { name: 'get_credits', arguments: {} });
  if (credits.credits.total < minRequired) {
    throw new Error(
      `Insufficient credits: ${credits.credits.total} remaining, need ${minRequired}. ` +
      `Buy more credits via x402 or wait for daily reset at ${credits.credits.daily.resetsAt}.`
    );
  }
  return credits.credits.total;
}

// Check before each major step
await ensureCredits(3); // Need at least 3 for build + test + polish
```

## Paid Features

Paid features require **any completed credit purchase**:
- Preview and production deployments (mainnet)
- Code downloads
- Custom domains
- Advanced AI model preferences

The system checks `hasUserEverPaid()` — if the wallet has any completed payment record, paid features are unlocked **permanently**.

### How to Unlock

Buy credits via x402 using `topup_credits` (USDC on Solana, no browser needed). A single purchase ($15 minimum) creates a payment record and unlocks all paid features. See [x402 Credit Top-Up](#x402-credit-top-up) below.

### Check Payment Status

```typescript
const credits = await mcpCall('tools/call', {
  name: 'get_credits',
  arguments: {},
});
// If add-on credits exist, the wallet has paid and features are unlocked
```

## x402 Credit Top-Up

Agents can buy credits with USDC on Solana — no browser needed. Uses the x402 payment protocol with the PayAI facilitator (free, no API keys required).

> A completed x402 credit purchase also unlocks all paid features (mainnet deployment, code downloads, custom domains) since the system only requires that the user has ever completed any payment.

### Critical Rules

These rules are **non-negotiable**. Violating any one causes `unexpected_verify_error`:

| DO | DON'T |
|----|-------|
| Use the **facilitator address** as `tx.feePayer` (from `accepts[0].extra.feePayer` in the 402 response) | Use your own wallet as fee payer |
| **Partially sign** with `tx.partialSign(keypair)` | Fully sign the transaction |
| Send the unsigned tx to the server — the **facilitator submits it on-chain** | Submit the transaction on-chain yourself |
| Encode as x402 v2 PaymentPayload (base64 JSON with `x402Version`, `scheme`, `network`, `payload.transaction`) | Send raw signature bytes or raw transaction bytes |
| Use the amount from `accepts[0].amount` directly (already in USDC atomic units as a string) | Multiply the amount by `1e6` (it's already in micro-USDC) |
| Use **`createTransferCheckedInstruction`** (includes mint + decimals) | Use `createTransferInstruction` (facilitator rejects it) |
| Include exactly **3 instructions**: `setComputeUnitLimit` (≤50000) + `setComputeUnitPrice` + `transferChecked` | Use 1 instruction or >3 instructions, or set compute limit >50000 |
| For REST: send payment as **`X-PAYMENT`** header (x402 v2 standard) | Use `PAYMENT-SIGNATURE` or other non-standard header names |
| Use legacy `Transaction` from `@solana/web3.js` | Use `VersionedTransaction` or `@solana/kit` (facilitator may reject) |

> **Note:** `x402-axios` does NOT work with Poof's topup endpoint — it fails on schema validation because Poof's network identifier (`solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`) isn't in its enum and `maxAmountRequired` is missing from the response. Use the manual approach below instead.

### Manual Payment Flow

Construct the payment using `@solana/web3.js`. Works for both MCP tool usage and direct REST calls:

```typescript
import { Connection, PublicKey, Transaction, ComputeBudgetProgram } from '@solana/web3.js';
import { createTransferCheckedInstruction, getAssociatedTokenAddress, TOKEN_PROGRAM_ID } from '@solana/spl-token';

const USDC_MINT = new PublicKey('EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v');
const connection = new Connection('https://api.mainnet-beta.solana.com');

// Step 1: Request payment requirements (returns 402 with facilitator info)
const requirementsResponse = await mcpCall('tools/call', {
  name: 'topup_credits',
  arguments: { quantity: 1 },
});

// Parse the 402 response body — the MCP tool returns it as JSON text
// Response shape:
// {
//   x402Version: 2,
//   accepts: [{
//     scheme: 'exact',
//     network: 'solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp',
//     amount: '15000000',          ← USDC atomic units (already multiplied, $15 = 15000000)
//     payTo: '<treasury-address>',  ← where USDC goes
//     asset: 'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v',
//     extra: {
//       feePayer: '2wKupLR9q6wXYppw8Gr2NvWxKBUqm4PPJKkQfoxHDBg4',  ← FACILITATOR address
//       ...
//     }
//   }],
//   priceUsd: 15,
//   priceUsdc: '15000000',
//   credits: 50,
//   quantity: 1,
// }
const paymentReqs = requirementsResponse.accepts[0];
const facilitatorAddress = paymentReqs.extra.feePayer;
const treasuryAddress = paymentReqs.payTo;
const amountMicroUsdc = Number(paymentReqs.amount); // Already in atomic units!
const network = paymentReqs.network;

// Step 2: Build a USDC transfer tx with the FACILITATOR as fee payer
// IMPORTANT: The facilitator requires EXACTLY 3 instructions in this order:
//   1. ComputeBudgetProgram.setComputeUnitLimit (≤ 50000 CU)
//   2. ComputeBudgetProgram.setComputeUnitPrice
//   3. createTransferCheckedInstruction (NOT createTransferInstruction)
const facilitatorPubkey = new PublicKey(facilitatorAddress);
const treasuryPubkey = new PublicKey(treasuryAddress);
const senderAta = await getAssociatedTokenAddress(USDC_MINT, keypair.publicKey);
const treasuryAta = await getAssociatedTokenAddress(USDC_MINT, treasuryPubkey);

const tx = new Transaction().add(
  ComputeBudgetProgram.setComputeUnitLimit({ units: 50000 }),
  ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 1000 }),
  createTransferCheckedInstruction(
    senderAta,         // from: your USDC token account
    USDC_MINT,         // mint: USDC mint address (required by transferChecked)
    treasuryAta,       // to: Poof treasury USDC token account
    keypair.publicKey, // authority: your wallet (signs the transfer)
    amountMicroUsdc,   // amount: already in atomic units from the 402 response
    6,                 // decimals: USDC has 6 decimals
    [],
    TOKEN_PROGRAM_ID
  )
);
tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
tx.feePayer = facilitatorPubkey;  // MUST be the facilitator — NOT your keypair
tx.partialSign(keypair);          // Sign ONLY as the token transfer authority
// DO NOT call connection.sendTransaction() — the facilitator submits it

// Step 3: Encode as x402 v2 PaymentPayload
const serializedTx = Buffer.from(
  tx.serialize({ requireAllSignatures: false }) // Allow missing facilitator signature
).toString('base64');

const x402Payment = Buffer.from(JSON.stringify({
  x402Version: 2,
  scheme: 'exact',
  network: network, // Use the network from the 402 response
  payload: {
    transaction: serializedTx,
  },
})).toString('base64');

// Step 4: Complete purchase — send the payment header back
const result = await mcpCall('tools/call', {
  name: 'topup_credits',
  arguments: {
    quantity: 1,
    payment: x402Payment,
  },
});
// Returns: { credits: 50, priceUsd: 15, txId: '...' }
```

**For direct REST calls** (instead of MCP), replace Steps 1 and 4 with fetch:

```typescript
// Step 1 (REST): Get payment requirements
const reqRes = await fetch(`${env.baseUrl}/api/credits/topup`, {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${idToken}`, 'X-Wallet-Address': walletAddress, 'Content-Type': 'application/json' },
  body: JSON.stringify({ quantity: 1 }),
});
const requirementsResponse = await reqRes.json(); // 402 response body

// ... Steps 2-3 same as above (build tx, encode payload) ...

// Step 4 (REST): Send payment via X-PAYMENT header
const result = await fetch(`${env.baseUrl}/api/credits/topup`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${idToken}`,
    'X-Wallet-Address': walletAddress,
    'X-PAYMENT': x402Payment,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ quantity: 1 }),
});
const data = await result.json(); // { credits: 50, priceUsd: 15, txId: '...' }
```

### How x402 Works (Under the Hood)

Understanding this prevents common mistakes:

```
Agent                          Poof Server                    PayAI Facilitator
  │                                │                                │
  │ POST /api/credits/topup        │                                │
  │ (no payment header)            │                                │
  │ ──────────────────────────────►│                                │
  │                                │  GET /supported                │
  │                                │ ──────────────────────────────►│
  │                                │  { feePayer: "2wKup..." }     │
  │                                │ ◄──────────────────────────────│
  │  402 { accepts: [...],         │                                │
  │    extra: { feePayer } }       │                                │
  │ ◄──────────────────────────────│                                │
  │                                │                                │
  │ Build tx:                      │                                │
  │   feePayer = facilitator       │                                │
  │   transfer USDC → treasury     │                                │
  │   partialSign(myKeypair)       │                                │
  │   DO NOT sendTransaction!      │                                │
  │                                │                                │
  │ POST /api/credits/topup        │                                │
  │ + X-PAYMENT header             │                                │
  │ ──────────────────────────────►│                                │
  │                                │  POST /verify                  │
  │                                │  { paymentPayload, reqs }      │
  │                                │ ──────────────────────────────►│
  │                                │  { isValid: true }             │
  │                                │ ◄──────────────────────────────│
  │                                │  POST /settle                  │
  │                                │  (facilitator co-signs &       │
  │                                │   submits tx on-chain)         │
  │                                │ ──────────────────────────────►│
  │                                │  { success, transaction }      │
  │                                │ ◄──────────────────────────────│
  │  200 { credits: 50, txId }     │                                │
  │ ◄──────────────────────────────│                                │
```

### Pricing

| Quantity | USDC | Credits | Expiry |
|----------|------|---------|--------|
| 1 package | $15 | 50 | 6 months |
| 5 packages | $75 | 250 | 6 months |
| 10 packages (max) | $150 | 500 | 6 months |

Any authenticated user can purchase. Payments are idempotent via transaction ID. A completed purchase also unlocks paid features (deployment, downloads, etc.).

### Troubleshooting x402 Payments

| Error | Cause | Fix |
|-------|-------|-----|
| `unexpected_verify_error` | Generic — the facilitator rejected the payment or the header couldn't be parsed | Check ALL the rules in the Critical Rules table above. Most common: wrong fee payer, tx already submitted, or bad encoding |
| `Invalid X-PAYMENT header format` | The header is not valid base64-encoded JSON with `x402Version` and `payload` fields | Ensure the header is `base64(JSON({ x402Version: 2, scheme: "exact", network: "...", payload: { transaction: "..." } }))` |
| `missing x402Version field` | The decoded JSON doesn't have `x402Version` | You may be sending raw tx bytes instead of the x402 PaymentPayload wrapper |
| `missing payload field` | The decoded JSON doesn't have `payload` | Wrap the serialized transaction inside `{ payload: { transaction: "..." } }` |
| `not valid base64 JSON` | The header is not valid base64 or doesn't decode to JSON | Double-check your base64 encoding. The ENTIRE PaymentPayload JSON must be base64-encoded |
| `Payment settlement failed` | Facilitator couldn't submit the tx on-chain | Ensure the tx is not already submitted, the blockhash is recent, and you have sufficient USDC balance |
| HTTP 402 returned twice | First call is expected to return 402 (payment requirements). Second call with payment header should return 200 | This is normal for the first call. Parse the 402 body and construct the payment |

**Debug checklist if you get `unexpected_verify_error`:**
1. Is `tx.feePayer` set to the facilitator address from `accepts[0].extra.feePayer`? (NOT your wallet)
2. Did you call `tx.partialSign(keypair)` and NOT `tx.sign(keypair)` or `connection.sendTransaction()`?
3. Is the tx serialized with `{ requireAllSignatures: false }`?
4. Is the serialized tx wrapped in `{ x402Version: 2, scheme: "exact", network: "...", payload: { transaction: "..." } }` and then base64-encoded?
5. Is the amount from `accepts[0].amount` used directly (NOT multiplied by `1e6`)?
6. Does your wallet have enough USDC balance for the transfer?

## AI Preferences

Control the intelligence tier used for different AI operations:

```typescript
// Get current preferences
const prefs = await mcpCall('tools/call', {
  name: 'get_ai_preferences',
  arguments: {},
});
// Returns: tier per use case (mainChat, planMode, codingAgent, etc.)

// Update preferences (requires any credit purchase)
await mcpCall('tools/call', {
  name: 'set_ai_preferences',
  arguments: {
    preferences: {
      mainChat: 'genius',      // 'average' | 'smart' | 'genius'
      codingAgent: 'smart',
    },
  },
});
```
