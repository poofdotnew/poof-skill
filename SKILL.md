---
name: poof
description: Use when building AI agents that interact with Poof (poof.new), creating or managing Solana dApps programmatically, using the Poof MCP endpoint or REST API, or connecting a local frontend to a Poof backend. Triggers: 'poof project', 'deploy dApp', 'create poof app', 'poof.new API', 'MCP tools for poof', 'Solana agent', 'blockchain app builder', '@pooflabs/server', '@pooflabs/web'.
---

# Poof Agent SDK

Build autonomous AI agents that create, deploy, and manage full-stack Solana applications on [poof.new](https://poof.new) — entirely programmatically, no browser needed.

## How It Works

Poof exposes a **Model Context Protocol (MCP)** endpoint at `POST /api/mcp` with 30 tools covering the complete project lifecycle. Your agent authenticates with a Cognito JWT, then drives everything through chat — the Poof AI handles code generation, database policies, deployments, and blockchain operations.

```
Your Agent ──► @pooflabs/server (init + getIdToken) ──► Cognito JWT
    │
    ▼
POST /api/mcp ──► create_project, chat, cancel_ai, steer_ai, get_files, publish_project, etc.
```

## Authentication

All Poof API calls require two headers:

```
Authorization: Bearer <cognito-id-token>
X-Wallet-Address: <solana-wallet-address>
```

### Keypair Setup

You need a Solana keypair to authenticate. If you don't already have one, generate it:

```typescript
import { Keypair } from '@solana/web3.js';
import bs58 from 'bs58';

// Generate a new keypair
const keypair = Keypair.generate();
const privateKey = bs58.encode(keypair.secretKey);
const walletAddress = keypair.publicKey.toBase58();

console.log('Private key (base58):', privateKey);
console.log('Wallet address:', walletAddress);

// Store these — you'll need them for all API calls
process.env.TAROBASE_SOLANA_KEYPAIR = privateKey;
process.env.SOLANA_WALLET_ADDRESS = walletAddress;
```

> Generate a fresh keypair for each agent. Only use an existing keypair if the user explicitly provides one.

### Get Auth Token

```typescript
import 'dotenv/config';
import { init, getIdToken } from '@pooflabs/server';

const POOF_ENVIRONMENTS = {
  production: {
    appId: '697d5189a1e3dd2cc1a82d2b',
    baseUrl: 'https://poof.new',
    authUrl: 'https://auth.tarobase.com',
  },
  staging: {
    appId: '6993d4b0b2b6ac08cd334dfb',
    baseUrl: 'https://staging.poof.new',
    authUrl: 'https://auth-staging.tarobase.com',
  },
  local: {
    appId: '6993d4b0b2b6ac08cd334dfb',
    baseUrl: 'http://localhost:3000',
    authUrl: 'https://auth-staging.tarobase.com',
  },
} as const;

type PoofEnv = keyof typeof POOF_ENVIRONMENTS;
const env = POOF_ENVIRONMENTS[(process.env.POOF_ENV as PoofEnv) || 'production'];

init({ appId: env.appId, authApiUrl: env.authUrl });
process.env.TAROBASE_SOLANA_KEYPAIR = process.env.SOLANA_PRIVATE_KEY!;

const walletAddress = process.env.SOLANA_WALLET_ADDRESS!;
const idToken = await getIdToken();
```

You only need `init` and `getIdToken` from the SDK. Everything else goes through the MCP `chat` tool.

> **Multi-wallet:** If your agent manages multiple wallets, use `createWalletClient({ keypair })` from `@pooflabs/server` after `init()` to create isolated clients. Each client has its own auth session and operates as its own wallet (with `set`, `get`, `signMessage`, `signTransaction`, etc. plus `.address`). Pass the keypair as a base58 string.

| Variable | Purpose | Required |
|----------|---------|----------|
| `SOLANA_PRIVATE_KEY` | Solana private key (base58) | Yes |
| `SOLANA_WALLET_ADDRESS` | Your Solana wallet public address | Yes |
| `POOF_ENV` | Target environment: `production` (default), `staging`, or `local` | No |

> **Important:** The keypair you use becomes the project owner. The wallet address is set as `ADMIN_ADDRESS` in your project's constants, controlling admin permissions in policies.

### Staging / Local Setup

Set `POOF_ENV` in your `.env` to switch environments. All auth credentials, app IDs, and URLs are configured automatically.

| `POOF_ENV` | Base URL | Use case |
|------------|----------|----------|
| `production` (default) | `https://poof.new` | Live agents |
| `staging` | `https://staging.poof.new` | Testing against staging |
| `local` | `http://localhost:3000` | Local dev server |

## Quick Workflow

See [docs/building-and-chat.md](docs/building-and-chat.md) for the full `mcpCall` helper, `pollUntilDone` with timeout/backoff, and `uuidv4` import.

```typescript
// 1. Auth
const idToken = await getIdToken();

// 2. Create project (AI starts building immediately)
// generationMode: 'full' (default) | 'policy' | 'ui,policy' | 'backend,policy'
// - 'full': Poof handles everything incl. deployment & custom domains
// - 'policy': You build your own app; Poof provides DB policies + typed SDK
// - 'ui,policy' / 'backend,policy': Poof generates one side, you build the other
// Almost every project needs 'policy' at minimum. See docs/building-and-chat.md for details.
await mcpCall('tools/call', {
  name: 'create_project',
  arguments: { projectId: uuidv4(), firstMessage: 'Build a ...', tarobaseToken: idToken, isPublic: true },
});

// 3. Poll until done (timeout + exponential backoff — see building-and-chat.md)
await pollUntilDone(projectId);

// 4. Iterate via chat
await mcpCall('tools/call', {
  name: 'chat',
  arguments: { projectId, message: 'Add a leaderboard', messageId: uuidv4(), tarobaseToken: idToken },
});

// 4b. Steer mid-execution (optional — redirect AI without cancelling)
await mcpCall('tools/call', {
  name: 'steer_ai',
  arguments: { projectId, message: 'Focus on the backend first, skip the UI for now' },
});

// 4c. Cancel if stuck (optional)
await mcpCall('tools/call', { name: 'cancel_ai', arguments: { projectId } });

// 4d. Read/modify files directly (optional — get_files requires a credit purchase)
const files = await mcpCall('tools/call', { name: 'get_files', arguments: { projectId } });
await mcpCall('tools/call', {
  name: 'update_files',
  arguments: { projectId, files: { 'src/config.ts': 'export const MAX = 100;' }, tarobaseToken: idToken },
});

// 5. Deploy
await mcpCall('tools/call', {
  name: 'publish_project',
  arguments: { projectId, target: 'production', authToken: idToken },
});
```

**Environment note:** After `create_project`, the app runs on **Draft (Poofnet)** — a simulated blockchain with fake tokens. This is free and great for testing. To test with real mainnet tokens, deploy to **Preview** using `publish_project` with `target: 'preview'`. See [docs/deployment.md](docs/deployment.md) for the full environment breakdown.

See [docs/building-and-chat.md](docs/building-and-chat.md) for the full workflow with `mcpCall` helper.

> **Token key names:** `tarobaseToken` is used for `create_project`, `chat`, `update_files`, and other project operations. `authToken` is the same token passed under a different key for `publish_project`. Both come from `getIdToken()`.

### Agent Workflow Checklist

Copy this checklist and track your progress:

```
- [ ] Auth: Generate keypair + get token via getIdToken()
- [ ] Create: create_project with firstMessage
- [ ] Poll: pollUntilDone(projectId)
- [ ] Verify: get_project_status, check draft URL
- [ ] Test: Generate + run lifecycle tests via chat
- [ ] Evaluate: get_test_results → check summary.failed === 0
- [ ] Fix: If tests failed, iterate with structured error details
- [ ] Deploy: security_scan → check_publish_eligibility → publish_project
```

## Documentation

Read these for deeper context — especially **how-poof-works** if you're orchestrating what the Poof AI should build.

| Doc | What it covers |
|-----|----------------|
| [**How Poof Works**](docs/how-poof-works.md) | Architecture, policy system, plugins, on-chain vs off-chain, what Poof can/can't do. **Read this to write effective prompts.** |
| [**Building & Chat**](docs/building-and-chat.md) | Project creation, chat workflow, polling, follow-up patterns, generation modes, full code example. |
| [**Backend-Only Mode**](docs/backend-only.md) | Using `backend,policy` generation mode with a local frontend — connection info, `@pooflabs/web` setup, PartyServer integration. |
| [**Local Frontend Guide**](docs/local-frontend-guide.md) | Building a frontend that connects to a Poof backend — SDK init, wallet auth, database access, API routes, real-time subscriptions, React hooks, gotchas. |
| [**Database SDK**](docs/database-sdk.md) | The generated db-client + collections pattern — typed functions, read/write, frontend vs backend, how to extract and use. |
| [**Deployment**](docs/deployment.md) | Environments (draft/preview/production/mobile), publishing, code downloads. |
| [**Credits & Payments**](docs/credits-and-payments.md) | Credit system, paid features, x402 USDC top-up flow. |
| [**Testing**](docs/testing.md) | Lifecycle actions, test files, bootstrap scripts, policy validation. |
| [**API Reference**](docs/api-reference.md) | All 30 MCP tools, REST endpoints, JSON-RPC format, error codes. |
| [**Troubleshooting**](docs/troubleshooting.md) | Common errors, recovery patterns, stuck build handling, credit exhaustion. |
| [**Cross-Compatibility**](docs/cross-compatibility.md) | curl examples, Python helper, OpenAI function calling format, direct REST usage. |

## Post-Build Verification

After the Poof AI finishes building (i.e., `check_ai_active` returns `false`), **don't assume it works correctly**. Always verify by running lifecycle actions and checking project status.

### 1. Run Lifecycle Tests

Ask the Poof AI to generate and run tests for the features it just built:

```typescript
// After initial build completes, send a follow-up to generate + run tests
await mcpCall('tools/call', {
  name: 'chat',
  arguments: {
    projectId,
    message: 'Generate lifecycle action tests for all the policies you just created. Test both success and failure cases — verify that authorized users can perform operations and unauthorized users are denied. Run the tests to confirm they pass.',
    messageId: uuidv4(),
    tarobaseToken: await getIdToken(),
  },
});

await pollUntilDone(projectId);
```

The Poof AI will:
- Generate `lifecycle-actions/test-*.json` files that validate your policies
- Execute them against ephemeral test environments
- Report pass/fail results

### 2. Check Project Status & Get URLs

```typescript
const status = await mcpCall('tools/call', {
  name: 'get_project_status',
  arguments: { projectId },
});
console.log('Draft URL:', status.urls?.draft);
```

### 3. Request Bootstrap Scripts (If Needed)

If your app needs initial data (configs, counters, default settings):

```typescript
await mcpCall('tools/call', {
  name: 'chat',
  arguments: {
    projectId,
    message: 'Create bootstrap scripts to initialize the app with default configuration and any required seed data.',
    messageId: uuidv4(),
    tarobaseToken: await getIdToken(),
  },
});
```

### Recommended Verification Loop

For agents building projects autonomously, use this pattern:

```typescript
// 1. Build the project
await mcpCall('tools/call', {
  name: 'create_project',
  arguments: { projectId, firstMessage: prompt, tarobaseToken, isPublic: true },
});
await pollUntilDone(projectId);

// 2. Generate and run tests
await mcpCall('tools/call', {
  name: 'chat',
  arguments: {
    projectId,
    message: 'Generate and run lifecycle action tests for all policies. Cover both allowed and denied operations.',
    messageId: uuidv4(),
    tarobaseToken: await getIdToken(),
  },
});
await pollUntilDone(projectId);

// 3. Check structured test results
const testResults = await mcpCall('tools/call', {
  name: 'get_test_results',
  arguments: { projectId },
});

// 4. Evaluate pass/fail using structured data
const allPassed = testResults.summary.total > 0 && testResults.summary.failed === 0 && testResults.summary.errors === 0;

// 5. If tests failed, send a targeted fix with structured error details
if (!allPassed) {
  const failedTests = testResults.results
    .filter((r: any) => r.status === 'failed' || r.status === 'error')
    .map((r: any) => `- ${r.fileName}: ${r.lastError || r.status}`)
    .join('\n');

  await mcpCall('tools/call', {
    name: 'chat',
    arguments: {
      projectId,
      message: `The following tests failed:\n\n${failedTests}\n\nPlease fix the failing tests and re-run them.`,
      messageId: uuidv4(),
      tarobaseToken: await getIdToken(),
    },
  });
  await pollUntilDone(projectId);
}
```

See [docs/testing.md](docs/testing.md) for details on lifecycle action syntax and patterns.

## Writing the `firstMessage` Prompt

The `firstMessage` you pass to `create_project` determines what the Poof AI builds. Structure it with:

1. **Feature descriptions** — what the app does, in natural language
2. **Data architecture** — collection paths, field names and types, on-chain vs off-chain
3. **Access rules** — who can read, create, update, delete
4. **Token operations** — which token, amounts, who pays whom
5. **UI requirements** — pages, layout, theme

**Don't hardcode plugin names** — describe the behavior (e.g., "bonding curve so the token is immediately tradeable") and let the Poof AI pick the right plugin. It knows its own ecosystem better than your agent does. Only name plugins when the user explicitly requests a specific one (e.g., "use Pump.fun" → `@PumpFunPlugin`).

See [docs/how-poof-works.md](docs/how-poof-works.md) for the architecture knowledge that makes prompts effective, and [docs/building-and-chat.md](docs/building-and-chat.md#writing-effective-prompts) for good vs weak prompt examples.

## Best Practices

- **Refresh tokens** — the `mcpCall` helper in [building-and-chat.md](docs/building-and-chat.md) refreshes on every call. If building your own client, refresh before each batch (~1 hour expiry)
- **One message at a time** — always `await pollUntilDone(projectId)` before sending the next `chat`. Sending while AI is active queues the message (FIFO), but the AI won't have your agent's evaluation context
- **Poll with `pollUntilDone`** after every `chat` and `create_project` — includes timeout + exponential backoff. See [building-and-chat.md](docs/building-and-chat.md#polling-helper)
- **Use `uuidv4()`** for all IDs (projectId, messageId) — import from the `uuid` package
- **Check credits before AND during long workflows** — `get_credits` is free. A full build + test + polish cycle costs 3-5 credits. If credits run out mid-build, the AI stops responding. Free tier gets ~10 daily credits
- **Any credit purchase unlocks deployment** — mainnet deployment requires that the wallet has completed at least one credit purchase. An x402 `topup_credits` purchase ($15 minimum) is the agent-friendly way to unlock both AI credits and deployment access without a browser. Once paid, paid features are permanently unlocked
- **Always verify after building** — generate lifecycle tests and run them before deploying
- **Deploy to preview first** — test before production
- **Run `security_scan` before production deploys** — catches policy and code vulnerabilities
- **Use `delete_project` to clean up** failed or abandoned projects
- **Understand how Poof works** — your prompts to `chat` are more effective when you know the policy system, plugin ecosystem, and on-chain/off-chain tradeoffs (see [how-poof-works](docs/how-poof-works.md))
- **Create a dedicated directory** for each agent script — use the [scaffolding template](docs/building-and-chat.md#scaffolding-template) with its own `package.json`, `.env`, `tsconfig.json`, and `src/index.ts`
- **Always generate a fresh Solana keypair** — never look for, read, or use the user's existing keypairs (e.g. `~/.config/solana/id.json`) unless they explicitly ask. Everything should be self-contained in the project directory
- **No rate limits** are currently enforced on MCP tool calls, but avoid excessive concurrent requests
