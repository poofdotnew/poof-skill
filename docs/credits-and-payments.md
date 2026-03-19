# Credits & Payments

## Contents
- [Credit System](#credit-system)
- [Membership](#membership)
- [x402 Credit Top-Up](#x402-credit-top-up)
- [AI Preferences](#ai-preferences)

## Credit System

Poof uses credits for AI interactions (chat messages, builds). Check your balance before starting long workflows:

```typescript
const credits = await mcpCall('tools/call', {
  name: 'get_credits',
  arguments: {},
});
// Returns: daily (free), subscription, and add-on credits with totals
```

### Credit Types

| Type | Source | Details |
|------|--------|---------|
| Daily | Free | ~10 credits, reset daily, available to all users |
| Subscription | Premium plan | Included with $30/month subscription |
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
      `Subscribe for more credits or wait for daily reset at ${credits.credits.daily.resetsAt}.`
    );
  }
  return credits.credits.total;
}

// Check before each major step
await ensureCredits(3); // Need at least 3 for build + test + polish
```

## Membership

```typescript
const membership = await mcpCall('tools/call', {
  name: 'get_membership',
  arguments: {},
});
// Returns: tier, status, period details
```

Premium ($30/month) is required for:
- Preview and production deployments
- Code downloads

Add-on credit purchases via x402 do **not** require a subscription — any authenticated user can buy credits.

### Subscribe

```typescript
const checkout = await mcpCall('tools/call', {
  name: 'subscribe',
  arguments: {},
});
// Returns: Stripe checkout URL — open in browser to complete
```

## x402 Credit Top-Up

Agents can buy credits with USDC on Solana — no browser needed. Uses the x402 payment protocol.

### Flow

```typescript
// Step 1: Get payment requirements
const requirements = await mcpCall('tools/call', {
  name: 'topup_credits',
  arguments: { quantity: 1 },
});
// Returns: treasury address, USDC amount, network info

// Step 2: Sign a USDC transfer tx with your Solana wallet
import { Connection, PublicKey, Transaction } from '@solana/web3.js';
import { createTransferInstruction, getAssociatedTokenAddress, TOKEN_PROGRAM_ID } from '@solana/spl-token';

const USDC_MINT = new PublicKey('EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v');
const connection = new Connection('https://api.mainnet-beta.solana.com');

const senderAta = await getAssociatedTokenAddress(USDC_MINT, keypair.publicKey);
const treasuryAta = await getAssociatedTokenAddress(USDC_MINT, new PublicKey(requirements.treasuryAddress));

const tx = new Transaction().add(
  createTransferInstruction(senderAta, treasuryAta, keypair.publicKey, requirements.usdcAmount * 1e6, [], TOKEN_PROGRAM_ID)
);
tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
tx.feePayer = keypair.publicKey;
tx.sign(keypair);

const base64EncodedSignature = Buffer.from(tx.serialize()).toString('base64');

// Step 3: Complete purchase with payment signature
const result = await mcpCall('tools/call', {
  name: 'topup_credits',
  arguments: {
    quantity: 1,
    paymentSignature: base64EncodedSignature,
  },
});
// Returns: { credits: 50, priceUsd: 15, txId: '...' }
```

### Pricing

| Quantity | USDC | Credits | Expiry |
|----------|------|---------|--------|
| 1 package | $15 | 50 | 6 months |
| 5 packages | $75 | 250 | 6 months |
| 10 packages (max) | $150 | 500 | 6 months |

No subscription required — any authenticated user can purchase. Payments are idempotent via transaction ID.

## AI Preferences

Control the intelligence tier used for different AI operations:

```typescript
// Get current preferences
const prefs = await mcpCall('tools/call', {
  name: 'get_ai_preferences',
  arguments: {},
});
// Returns: tier per use case (mainChat, planMode, codingAgent, etc.)

// Update preferences (requires subscription)
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
