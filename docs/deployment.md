# Deployment

Poof projects have four environments. Your agent can deploy to any of them via MCP.

## Contents
- [Environments](#environments)
- [Pre-Deploy Checks](#pre-deploy-checks)
- [Publishing](#publishing)
- [Code Downloads](#code-downloads)
- [Custom Domains](#custom-domains)
- [Authentication on Deployed Apps](#authentication-on-deployed-apps)

## Environments

Poof has four distinct environments. Understanding the difference is critical — especially between Draft and Preview.

| Environment | Also called | Blockchain | URL pattern | Cost |
|-------------|------------|-----------|-------------|------|
| **Draft** | Poofnet | **Simulated** (fake tokens, free) | `{slug}-preview.poof.new` | Free |
| **Preview** | Mainnet Preview | **Real mainnet** (real tokens) | `{slug}-mainnet-preview.poof.new` | Any payment |
| **Production** | Live | **Real mainnet** (real tokens) | `*.poof.new` or custom domain | Any payment |
| **Mobile** | — | **Real mainnet** (real tokens) | Seeker / iOS / Android | Any payment |

### Draft (Poofnet) vs Preview (Mainnet Preview)

This is the most common source of confusion:

- **Draft / Poofnet** — Automatically available for every project, no deployment needed. Uses a **simulated blockchain** — all token transfers, staking, swaps etc. use fake/test tokens with zero real cost. Great for rapid iteration and testing. Token balances are simulated and won't reflect real amounts.

- **Preview / Mainnet Preview** — Requires explicit deployment via `publish_project`. Uses the **real Solana mainnet** — transactions cost real SOL, tokens are real. This is your staging environment to test with real blockchain behavior before going to production.

**Key rule:** If you're just building and iterating, you're on Draft (Poofnet). Token balances are fake. When you deploy to Preview or Production, tokens become real.

Draft is always available — no deployment needed. Preview and production require that the user has **completed at least one credit purchase** via x402.

> **How it works:** The system calls `hasUserEverPaid()` — if the wallet has any completed payment record, deployment is unlocked permanently. If `check_publish_eligibility` returns `no_membership`, buy credits via `topup_credits` ($15 minimum for 50 credits) — this both provides AI credits and unlocks deployment.

## Pre-Deploy Checks

### Security Scan

Run a security audit before deploying to production:

```typescript
const scan = await mcpCall('tools/call', {
  name: 'security_scan',
  arguments: { projectId, tarobaseToken },
});
```

### Eligibility Check

Always check eligibility before deploying:

```typescript
const check = await mcpCall('tools/call', {
  name: 'check_publish_eligibility',
  arguments: { projectId },
});
// Returns: payment status, security review status, project readiness
```

## Publishing

```typescript
// Deploy to preview (recommended first)
await mcpCall('tools/call', {
  name: 'publish_project',
  arguments: {
    projectId,
    target: 'preview',       // 'preview' | 'production' | 'mobile'
    authToken: tarobaseToken,
  },
});
```

Deployment creates a task. Poll it to track progress:

```typescript
const tasks = await mcpCall('tools/call', {
  name: 'list_tasks',
  arguments: { projectId },
});
// tasks is an object with a tasks array — access via tasks.tasks
const latestTask = tasks.tasks?.[0];

const task = await mcpCall('tools/call', {
  name: 'get_task',
  arguments: { projectId, taskId: latestTask.id },
});
// Poll until task.status is 'completed'
```

### Deployment Flow

1. **Deploy to preview first** — test with real mainnet behavior
2. **Verify on preview** — check functionality, transactions, UI
3. **Deploy to production** — when preview looks good
4. **Deploy to mobile** (optional) — for Seeker, iOS, or Android

## Static Frontend Deploy

If you're building your frontend outside of Poof (e.g. with `generationMode: 'backend,policy'` or `'policy'`) and want to deploy it to your Poof project, use the static deploy API. This uploads a pre-built `tar.gz` of your dist folder and hosts it on Poof alongside your backend.

Three ways to deploy:
- **REST API** — presigned URL flow (`upload-url` → S3 PUT → `trigger`)
- **MCP tool** — `deploy_static_frontend` with base64-encoded archive
- **CLI** — `poof deploy static -p <id> --archive dist.tar.gz`

See [static-deploy.md](static-deploy.md) for the full guide, API reference, and examples.

## Code Downloads

Export your project's source code:

```typescript
// 1. Initiate download
const download = await mcpCall('tools/call', {
  name: 'download_code',
  arguments: { projectId },
});

// 2. Poll the task until complete
let task;
do {
  await new Promise(r => setTimeout(r, 3000));
  task = await mcpCall('tools/call', {
    name: 'get_task',
    arguments: { projectId, taskId: download.taskId },
  });
} while (task.status !== 'completed');

// 3. Get signed download URL (expires in 5 minutes)
const url = await mcpCall('tools/call', {
  name: 'get_download_url',
  arguments: { projectId, taskId: download.taskId },
});
```

Requires at least one completed credit purchase.

## Custom Domains

After deploying to production, you can add custom domains:

```typescript
// List existing domains
const domains = await mcpCall('tools/call', {
  name: 'get_domains',
  arguments: { projectId },
});

// Add a custom domain
await mcpCall('tools/call', {
  name: 'add_domain',
  arguments: {
    projectId,
    domain: 'myapp.com',
    isDefault: true,
    tarobaseToken,
  },
});
```

After adding a domain, configure your DNS to point to Poof's servers. The response includes the required DNS records.

## Authentication on Deployed Apps

Deployed apps need wallet authentication for users:

- **Phantom Connect (recommended)** — self-service, supports all Solana wallets + social login, free, works on custom domains immediately
- **Privy** — works on `poof.new` subdomains automatically; custom domains require contacting the Poof team
