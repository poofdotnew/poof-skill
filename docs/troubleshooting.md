# Troubleshooting

Common errors, causes, and agent recovery patterns.

## Contents
- [Authentication Errors](#authentication-errors)
- [Build & Chat Errors](#build--chat-errors)
- [Deployment Errors](#deployment-errors)
- [Credit Errors](#credit-errors)
- [Agent Recovery Patterns](#agent-recovery-patterns)

## Authentication Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Not authenticated` | Invalid or expired JWT | Call `getIdToken()` again — tokens expire after ~1 hour |
| `Invalid projectId format` | Non-UUID project ID | Use `uuidv4()` from the `uuid` package |
| Auth succeeds but operations fail | Environment mismatch | Check `POOF_ENV` — must be `production`, `staging`, or `local`. Omit for production (default) |

## Build & Chat Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `check_ai_active` returns `true` for >10 minutes | AI is stuck or in a loop | Call `cancel_ai`, then send a new `chat` message with clearer instructions |
| `check_ai_active` returns `false` immediately after `chat` | AI server may be unreachable | Check the `status` field: `"error"` means the server is down (vs `"ok"` meaning AI genuinely finished). Retry after a short delay, or check `get_project_status` for the latest task status |
| `chat` returns error after build started | Token expired mid-build | Refresh token with `getIdToken()` and retry |
| AI builds the wrong thing | Vague or ambiguous `firstMessage` | Be specific about data models, access rules, token operations, and on-chain vs off-chain. See [how-poof-works.md](how-poof-works.md) |
| `Project not found` | Wrong project ID or not the owner | Verify project ID and that you're using the same keypair that created the project |

## Deployment Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `check_publish_eligibility` returns `no_membership` | User has never purchased credits | The wallet needs at least one completed credit purchase. Buy x402 add-on credits via `topup_credits` ($15 minimum). Once any purchase completes, deployment is permanently unlocked for that wallet |
| `check_publish_eligibility` fails with security review | Failed security scan | Run `security_scan` and fix flagged issues before retrying |
| Preview deploy fails | Missing `authToken` | Pass `authToken: await getIdToken()` in `publish_project` arguments |
| Production deploy fails | Haven't passed eligibility check | Run `check_publish_eligibility` first and resolve any blockers |
| `topup_credits` returns HTTP 500 | Settlement or credit award failed | Check `get_credits` to see if the payment was partially processed. If charged on-chain but no credits, contact support with the txId |

## x402 Payment Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `unexpected_verify_error` | The facilitator rejected the payment — usually wrong fee payer, tx already submitted, or bad encoding | See debug checklist in [credits-and-payments.md](credits-and-payments.md#troubleshooting-x402-payments) |
| `Invalid PAYMENT-SIGNATURE header format` | Header is not valid base64-encoded JSON with required x402 fields | Must be `base64(JSON({ x402Version: 2, scheme, network, payload: { transaction } }))` |
| `missing x402Version field` | Sending raw tx bytes instead of the x402 PaymentPayload wrapper | Wrap serialized tx inside the full PaymentPayload JSON structure |
| `not valid base64 JSON` | Header is not base64 or doesn't decode to valid JSON | Double-check encoding — the ENTIRE PaymentPayload JSON must be base64-encoded |
| `Payment settlement failed` | Facilitator couldn't submit the tx on-chain | Ensure tx is NOT already submitted, blockhash is recent, and wallet has sufficient USDC |
| First `topup_credits` call returns error (isError: true) | **This is expected** — the first call returns 402 with payment requirements | Parse the response body to get `accepts[0].extra.feePayer`, `accepts[0].payTo`, and `accepts[0].amount`, then construct the payment |

**Most common mistake:** Using your own wallet as `tx.feePayer`. The fee payer MUST be the facilitator address from `accepts[0].extra.feePayer` in the 402 response. The facilitator co-signs and submits the transaction — your agent should NEVER call `connection.sendTransaction()`.

## Credit Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `You have run out of credits` | No credits remaining | Check `get_credits` — wait for daily reset or top up via x402 |
| AI stops responding mid-build | Credits exhausted during execution | Check credits with `get_credits`. If zero, top up and send a new `chat` message to continue |
| `chat` silently does nothing | Insufficient credits to start | Always call `get_credits` before starting a workflow. A full build + test + polish cycle costs 3-5 credits |
| Agent can't deploy after buying credits | Payment may not have completed successfully | Verify the payment actually completed — check `get_credits` for add-on records. If the x402 call returned an error, follow the debug checklist in [credits-and-payments.md](credits-and-payments.md#troubleshooting-x402-payments) |

## Agent Recovery Patterns

### Stuck Build Recovery
```typescript
const status = await mcpCall('tools/call', {
  name: 'check_ai_active',
  arguments: { projectId },
});

if (status.active) {
  await mcpCall('tools/call', { name: 'cancel_ai', arguments: { projectId } });
  await new Promise(r => setTimeout(r, 2000));

  await mcpCall('tools/call', {
    name: 'chat',
    arguments: {
      projectId,
      message: 'The previous build got stuck. Please review the current state and continue from where you left off.',
      messageId: uuidv4(),
      tarobaseToken: await getIdToken(),
    },
  });
  await pollUntilDone(projectId);
}
```

### Credit Check Before Workflow
```typescript
async function ensureCredits(minRequired: number) {
  const credits = await mcpCall('tools/call', { name: 'get_credits', arguments: {} });
  if (credits.credits.total < minRequired) {
    throw new Error(
      `Insufficient credits: ${credits.credits.total} remaining, need ${minRequired}. ` +
      `Buy credits via x402 or wait for daily reset at ${credits.credits.daily.resetsAt}.`
    );
  }
}

await ensureCredits(5);
```

### Token Refresh Pattern
```typescript
async function withFreshToken<T>(fn: (token: string) => Promise<T>): Promise<T> {
  const token = await getIdToken();
  return fn(token);
}

await withFreshToken(token =>
  mcpCall('tools/call', {
    name: 'chat',
    arguments: { projectId, message: '...', messageId: uuidv4(), tarobaseToken: token },
  })
);
```
