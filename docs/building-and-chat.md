# Building & Chat

The `chat` tool is the primary way your agent interacts with Poof projects. It sends natural language instructions to the Poof AI, which then generates code, policies, UI, and deployments.

## Contents
- [MCP Helper](#mcp-helper)
- [Polling Helper](#polling-helper)
- [Project Lifecycle](#project-lifecycle)
- [Writing Effective Prompts](#writing-effective-prompts)
- [Updating Project Settings](#updating-project-settings)
- [Scaffolding Template](#scaffolding-template)
- [End-to-End Example](#end-to-end-example)

## MCP Helper

Use this helper for all MCP calls. You'll need a Solana keypair — see [SKILL.md](../SKILL.md#keypair-setup) for how to generate one if you don't have one.

```typescript
import 'dotenv/config';
import { init, getIdToken } from '@pooflabs/server';
import { v4 as uuidv4 } from 'uuid';

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

async function mcpCall(method: string, params?: any) {
  const idToken = await getIdToken();
  const res = await fetch(`${env.baseUrl}/api/mcp`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${idToken}`,
      'X-Wallet-Address': walletAddress,
    },
    body: JSON.stringify({ jsonrpc: '2.0', id: Date.now(), method, params }),
  });

  const text = await res.text();
  if (!res.ok) {
    throw new Error(`HTTP ${res.status}: ${text}`);
  }

  let json: any;
  try {
    json = JSON.parse(text);
  } catch {
    throw new Error(`Invalid JSON response: ${text.slice(0, 500)}`);
  }

  if (json.error) throw new Error(json.error.message);
  if (json.result?.isError) {
    throw new Error(json.result.content?.[0]?.text || 'Unknown MCP error');
  }

  const content = json.result?.content?.[0]?.text;
  if (!content) return json.result;

  try {
    return JSON.parse(content);
  } catch {
    return content;
  }
}
```

> **Dependencies:** Make sure your `package.json` includes `dotenv`, `@pooflabs/server`, `@solana/web3.js`, `bs58`, and `uuid`.

## Polling Helper

Use this helper for all polling. Includes timeout and exponential backoff to prevent hangs:

```typescript
async function pollUntilDone(projectId: string, maxMs = 600_000) {
  let delay = 5_000;
  const start = Date.now();
  while (Date.now() - start < maxMs) {
    await new Promise(r => setTimeout(r, delay));
    const status = await mcpCall('tools/call', {
      name: 'check_ai_active',
      arguments: { projectId },
    });
    if (!status.active && status.status !== 'error') return;
    if (status.status === 'error') {
      delay = Math.min(delay * 2, 30_000);
      continue;
    }
    delay = Math.min(delay * 1.5, 30_000);
  }
  await mcpCall('tools/call', { name: 'cancel_ai', arguments: { projectId } });
  throw new Error(`Build timed out after ${maxMs / 1000}s — cancelled via cancel_ai`);
}
```

## Project Lifecycle

### 1. Create a Project

```typescript
const projectId = uuidv4();
const tarobaseToken = await getIdToken();

await mcpCall('tools/call', {
  name: 'create_project',
  arguments: {
    projectId,
    firstMessage: 'Build a token-gated content platform with USDC payments',
    tarobaseToken,
    isPublic: true,
  },
});
```

The `firstMessage` is critical — it's the initial prompt that tells the Poof AI what to build. Be specific about features, data models, and blockchain operations. The AI starts building immediately.

### Generation Mode

Control what the AI generates with `generationMode`. Values are comma-separated components:

| Mode | What gets generated |
|------|-------------------|
| `full` (default) | Everything — UI, backend, policies, lifecycle actions |
| `policy` | Database policies only (schema, rules, hooks, queries) + lifecycle actions |
| `ui,policy` | Frontend UI + policies (no backend API routes) |
| `backend,policy` | Backend API routes + policies (no frontend UI) |

#### Choosing the right mode

- **`full`** — Best when you want Poof to handle everything end-to-end, including deployment and custom domain setup. Use this for standalone web apps where Poof owns the entire stack.
- **`policy`** — Use when you're building your own frontend or backend locally (e.g., a React Native app, a Next.js site, a custom server) but want Poof for the database layer. Almost every project needs policies, so this is the most common non-full mode. After creation, download the generated typed SDK via `download_code` → `get_download_url` and extract the DB client + collections into your own project.
- **`ui,policy`** — Use when you want Poof to generate the frontend and database policies, but you handle backend/API routes yourself (e.g., you already have a server or use serverless functions).
- **`backend,policy`** — Use when you're building your own frontend locally but want Poof to generate backend API routes and database policies. Good for React Native or desktop apps that need a hosted backend. After creation, call `get_project_status` to get `connectionInfo` (tarobaseAppId, backendUrl) for connecting your local frontend via `@pooflabs/web`. See [backend-only.md](backend-only.md) for the full workflow.

#### Auth requirement for policies

If your app uses policies with any access rules beyond `true` (i.e., rules that check the caller's identity), your frontend **must** integrate Tarobase login via the `@pooflabs/web` SDK so that writes are wallet-signed and the policy engine can verify the caller. Without this, the policy engine has no authenticated identity to evaluate rules against, and writes will be rejected.

The only exception is policies where all write rules are set to `true` (fully public access) — those don't require authentication since no identity check is performed.

```typescript
// Policy-only project (no UI, no backend)
await mcpCall('tools/call', {
  name: 'create_project',
  arguments: {
    projectId,
    firstMessage: 'Create a staking vault policy with 30-day lock period',
    tarobaseToken,
    generationMode: 'policy',
  },
});
```

You can change `generationMode` later via `update_project`.

### 2. Wait for the AI to Finish

Builds are async. Use the `pollUntilDone` helper (defined above) which includes timeout and exponential backoff:

```typescript
await pollUntilDone(projectId);
```

### 3. Check Project Status & Get URLs

After the build completes, always fetch the project status:

```typescript
const status = await mcpCall('tools/call', {
  name: 'get_project_status',
  arguments: { projectId },
});

console.log('Draft URL:', status.urls?.draft);
console.log('Editor URL:', `${env.baseUrl}/project/${projectId}`);
```

The draft URL runs on Poofnet (simulated blockchain, free). The editor URL (`poof.new/project/{projectId}`) is also always available with a built-in preview. See [deployment.md](deployment.md) for details.

### Execution Control

If the AI gets stuck or goes in the wrong direction, you have two options:

```typescript
// Option 1: Steer mid-execution (AI incorporates your message without stopping)
await mcpCall('tools/call', {
  name: 'steer_ai',
  arguments: {
    projectId,
    message: 'Focus on the backend policies, skip the UI for now',
  },
});

// Option 2: Cancel and start fresh
await mcpCall('tools/call', { name: 'cancel_ai', arguments: { projectId } });
// Then send a new chat message
```

### Direct File Editing

For quick fixes without going through chat, modify files directly:

```typescript
// Read current files
const result = await mcpCall('tools/call', {
  name: 'get_files',
  arguments: { projectId },
});

// Modify specific files
await mcpCall('tools/call', {
  name: 'update_files',
  arguments: {
    projectId,
    files: {
      'src/config.ts': 'export const API_URL = "https://api.example.com";',
    },
    tarobaseToken,
  },
});
```

### 4. Iterate via Chat

Send follow-up messages to refine the project:

```typescript
await mcpCall('tools/call', {
  name: 'chat',
  arguments: {
    projectId,
    message: 'Add a leaderboard page showing top contributors by USDC spent',
    messageId: uuidv4(),
    tarobaseToken,
  },
});

// Always poll after chat
// ... poll check_ai_active ...
```

Each `chat` call needs a unique `messageId` (UUID) and a fresh `tarobaseToken`.

> **One message at a time.** Always `await pollUntilDone(projectId)` before sending the next `chat` message. If you send while the AI is still active, the message gets queued (HTTP 202, `queued: true` in response). Queued messages execute in FIFO order, but the AI won't have context from your agent's evaluation of previous results. The pattern is always: `chat` → `pollUntilDone` → evaluate results → `chat` → `pollUntilDone`.

### 5. Get Conversation History

```typescript
const messages = await mcpCall('tools/call', {
  name: 'get_messages',
  arguments: { projectId },
});
```

Returns an object with a `messages` array (not a bare array). Each message has `role` (`"user"` or `"assistant"`) and `content` (string). Always access via `messages.messages`:

```typescript
const result = await mcpCall('tools/call', {
  name: 'get_messages',
  arguments: { projectId },
});
const msgList = result.messages || [];
```

## Writing Effective Prompts

Since `chat` is your main interface, the quality of your prompts matters. Knowing [how Poof works](how-poof-works.md) helps you write better ones.

### Good Prompts (Specific, Poof-Aware)

- "Add a staking vault where users lock SOL for 30 days. Use @AccountPlugin for the escrow PDA. On-chain with passthrough for the unstake operation."
- "Create a tipping system using passthrough — no stored records, just direct USDC transfers from tipper to creator."
- "Add user profiles (off-chain) with name, bio, and avatar. Owner-only write access. Public reads."
- "Add verifiable coin flip gambling with VRF. 2x payout on win, SOL stakes."

### Weak Prompts (Vague, May Miss Optimizations)

- "Add staking" — unclear what token, what duration, what mechanism
- "Add a database" — Poof IS the database; be specific about what data
- "Make it work on Ethereum" — impossible; Solana only

### Tips

1. **Specify on-chain vs off-chain** when it matters for cost/privacy
2. **Name the plugins** if you know the right one (e.g., "@DeFiPlugin for Meteora bonding curve")
3. **Describe access patterns** — who can read, who can write
4. **Be explicit about token operations** — which token, amounts, who pays whom
5. **Mention passthrough** for operations that don't need persistent storage

## Updating Project Settings

```typescript
await mcpCall('tools/call', {
  name: 'update_project',
  arguments: {
    projectId,
    updates: {
      title: 'My Token Platform',
      description: 'A token-gated content platform',
      isPublic: true,
    },
  },
});
```

Updates title, description, slug, visibility, generation mode, etc.

## Scaffolding Template

Minimal project structure for an agent script. Create a dedicated directory with these files:

**package.json:**
```json
{
  "name": "poof-agent",
  "type": "module",
  "scripts": { "start": "tsx src/index.ts" },
  "dependencies": {
    "@pooflabs/server": "latest",
    "@solana/web3.js": "^1.95.0",
    "bs58": "^6.0.0",
    "dotenv": "^16.0.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": { "tsx": "^4.0.0", "typescript": "^5.0.0" }
}
```

**tsconfig.json:**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "strict": true,
    "outDir": "dist"
  },
  "include": ["src"]
}
```

**.env:**
```
SOLANA_PRIVATE_KEY=<base58-encoded-private-key>
SOLANA_WALLET_ADDRESS=<public-key>
# POOF_ENV=staging  # uncomment for staging/local testing
```

Then put your agent logic in `src/index.ts` using the `mcpCall` and `pollUntilDone` helpers above.

## End-to-End Example

Complete standalone `src/index.ts` showing create → poll → test → verify → deploy:

```typescript
import 'dotenv/config';
import { init, getIdToken } from '@pooflabs/server';
import { v4 as uuidv4 } from 'uuid';

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

async function mcpCall(method: string, params?: any) {
  const idToken = await getIdToken();
  const res = await fetch(`${env.baseUrl}/api/mcp`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${idToken}`,
      'X-Wallet-Address': walletAddress,
    },
    body: JSON.stringify({ jsonrpc: '2.0', id: Date.now(), method, params }),
  });
  const json: any = await res.json();
  if (!res.ok || json.error) throw new Error(json.error?.message || `HTTP ${res.status}`);
  if (json.result?.isError) throw new Error(json.result.content?.[0]?.text || 'MCP error');
  const text = json.result?.content?.[0]?.text;
  if (!text) return json.result;
  try { return JSON.parse(text); } catch { return text; }
}

async function pollUntilDone(projectId: string, maxMs = 600_000) {
  let delay = 5_000;
  const start = Date.now();
  while (Date.now() - start < maxMs) {
    await new Promise(r => setTimeout(r, delay));
    const s = await mcpCall('tools/call', { name: 'check_ai_active', arguments: { projectId } });
    if (!s.active && s.status !== 'error') return;
    if (s.status === 'error') { delay = Math.min(delay * 2, 30_000); continue; }
    delay = Math.min(delay * 1.5, 30_000);
  }
  await mcpCall('tools/call', { name: 'cancel_ai', arguments: { projectId } });
  throw new Error(`Timed out after ${maxMs / 1000}s`);
}

async function main() {
  // 1. Check credits
  const credits = await mcpCall('tools/call', { name: 'get_credits', arguments: {} });
  if (credits.credits.total < 5) throw new Error(`Need 5+ credits, have ${credits.credits.total}`);
  console.log(`Credits: ${credits.credits.total}`);

  // 2. Create project
  const projectId = uuidv4();
  const tarobaseToken = await getIdToken();
  await mcpCall('tools/call', {
    name: 'create_project',
    arguments: {
      projectId,
      firstMessage: 'Build a token-gated content platform where creators post articles and readers pay 0.1 USDC per article. User profiles (off-chain), articles (off-chain), passthrough payment (on-chain).',
      tarobaseToken,
      isPublic: true,
    },
  });
  console.log(`Project created: ${projectId}`);

  // 3. Wait for build
  await pollUntilDone(projectId);
  console.log('Build complete');

  // 4. Generate and run tests
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

  // 5. Check structured test results
  const testResults = await mcpCall('tools/call', {
    name: 'get_test_results',
    arguments: { projectId },
  });
  const allPassed = testResults.summary.total > 0 && testResults.summary.failed === 0 && testResults.summary.errors === 0;

  if (!allPassed) {
    const failedTests = testResults.results
      .filter((r: any) => r.status === 'failed' || r.status === 'error')
      .map((r: any) => `- ${r.fileName}: ${r.lastError || r.status}`)
      .join('\n');
    console.log('Tests had failures — sending fix request');
    await mcpCall('tools/call', {
      name: 'chat',
      arguments: {
        projectId,
        message: `The following tests failed:\n\n${failedTests}\n\nPlease fix the issues and re-run the tests.`,
        messageId: uuidv4(),
        tarobaseToken: await getIdToken(),
      },
    });
    await pollUntilDone(projectId);
  }

  // 6. Get URLs (always use what the API returns — never construct URLs)
  const status = await mcpCall('tools/call', { name: 'get_project_status', arguments: { projectId } });
  console.log('Draft URL:', status.urls?.draft);
  console.log('Editor URL:', `${env.baseUrl}/project/${projectId}`);

  // 7. Deploy to preview (requires any credit purchase)
  const credits = await mcpCall('tools/call', { name: 'get_credits', arguments: {} });
  const hasPaid = credits.credits?.addOn?.purchased > 0;
  if (hasPaid) {
    await mcpCall('tools/call', {
      name: 'security_scan',
      arguments: { projectId, tarobaseToken: await getIdToken() },
    });
    await mcpCall('tools/call', {
      name: 'publish_project',
      arguments: { projectId, target: 'preview', authToken: await getIdToken() },
    });
    console.log('Deployed to preview');
  } else {
    console.log('Skipping deploy — credit purchase required. Use topup_credits to buy credits and unlock deployment.');
  }
}

main().catch(console.error);
```
