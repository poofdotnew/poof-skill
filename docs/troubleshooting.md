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
| `check_publish_eligibility` fails | Missing Premium subscription or failed security review | Subscribe via `subscribe` tool, run `security_scan` and fix issues |
| Preview deploy fails | Missing `authToken` | Pass `authToken: await getIdToken()` in `publish_project` arguments |
| Production deploy fails | Haven't passed eligibility check | Run `check_publish_eligibility` first and resolve any blockers |

## Credit Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `You have run out of credits` | No credits remaining | Check `get_credits` — wait for daily reset, subscribe for more, or top up via x402 |
| AI stops responding mid-build | Credits exhausted during execution | Check credits with `get_credits`. If zero, top up and send a new `chat` message to continue |
| `chat` silently does nothing | Insufficient credits to start | Always call `get_credits` before starting a workflow. A full build + test + polish cycle costs 3-5 credits |

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
      `Top up via x402 or wait for daily reset at ${credits.credits.daily.resetsAt}.`
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
