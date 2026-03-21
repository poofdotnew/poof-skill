<p align="center">
  <img src="https://poof.new/_next/image?url=%2Fimages%2Flogo-transparent-bg.png&w=256&q=75" alt="Poof" width="120" />
</p>

<h3 align="center">Poof Agent SDK</h3>

<p align="center">
  Build autonomous AI agents that create, deploy, and manage full-stack Solana dApps — entirely programmatically.
</p>

<p align="center">
  <a href="https://poof.new">Website</a> ·
  <a href="#quick-start">Quick Start</a> ·
  <a href="docs/api-reference.md">API Reference</a> ·
  <a href="docs/troubleshooting.md">Troubleshooting</a>
</p>

---

## What is this?

This repository contains the **Poof skill** — a documentation and reference package that teaches AI agents (Claude Code, Cursor, Windsurf, etc.) how to use the [Poof](https://poof.new) platform programmatically via its MCP API.

Poof is a Backend-as-a-Service for Solana. It generates full-stack dApps (database policies, backend APIs, frontend UI, blockchain operations) from natural language — and this skill gives your AI agent the knowledge to drive that entire lifecycle without a browser.

> **For AI agents:** The skill definition lives in [`SKILL.md`](SKILL.md). Detailed docs are in [`docs/`](docs/).

## How It Works

```
Your Agent ──► @pooflabs/server (auth) ──► Cognito JWT
    │
    ▼
POST /api/mcp ──► 30 tools: create, chat, deploy, test, download, and more
```

Your agent authenticates with a Solana keypair, gets a JWT, then calls Poof's MCP endpoint to create projects, iterate via chat, run tests, and deploy — all through code.

## Quick Start

### 1. Install dependencies

```bash
mkdir my-poof-agent && cd my-poof-agent
npm init -y
npm install @pooflabs/server @solana/web3.js bs58 dotenv uuid
npm install -D typescript @types/node
```

### 2. Set up environment

Create a `.env` file:

```env
SOLANA_PRIVATE_KEY=<your-base58-private-key>
SOLANA_WALLET_ADDRESS=<your-solana-public-key>
POOF_ENV=production
```

Don't have a keypair? Generate one:

```typescript
import { Keypair } from '@solana/web3.js';
import bs58 from 'bs58';

const keypair = Keypair.generate();
console.log('Private key:', bs58.encode(keypair.secretKey));
console.log('Wallet:', keypair.publicKey.toBase58());
```

### 3. Create your first project

```typescript
import 'dotenv/config';
import { init, getIdToken } from '@pooflabs/server';
import { v4 as uuidv4 } from 'uuid';

// Auth
init({
  appId: '697d5189a1e3dd2cc1a82d2b',
  authApiUrl: 'https://auth.tarobase.com',
});
process.env.TAROBASE_SOLANA_KEYPAIR = process.env.SOLANA_PRIVATE_KEY!;

const idToken = await getIdToken();
const projectId = uuidv4();

// Create project — Poof AI starts building immediately
await mcpCall('tools/call', {
  name: 'create_project',
  arguments: {
    projectId,
    firstMessage: 'Build a token-gated voting app where holders can create proposals and vote',
    tarobaseToken: idToken,
    isPublic: true,
  },
});

// Wait for build to finish, then deploy
await pollUntilDone(projectId);
await mcpCall('tools/call', {
  name: 'publish_project',
  arguments: { projectId, target: 'preview', authToken: idToken },
});
```

See [`docs/building-and-chat.md`](docs/building-and-chat.md) for the full `mcpCall` and `pollUntilDone` helper implementations.

### 4. Iterate

```typescript
// Chat to add features
await mcpCall('tools/call', {
  name: 'chat',
  arguments: {
    projectId,
    message: 'Add a leaderboard page showing top voters',
    messageId: uuidv4(),
    tarobaseToken: await getIdToken(),
  },
});
await pollUntilDone(projectId);
```

## Using as an AI Skill

This repo is designed to be consumed as a **skill** by AI coding agents.

### Claude Code

#### Option A: Install to your project (recommended)

Clone into your project's `.claude/skills/` directory — this makes it available whenever you work in that project and is shareable via git:

```bash
cd your-project
git clone https://github.com/poofdotnew/poof-skill.git .claude/skills/poof
```

#### Option B: Install globally

Clone to your home directory to make the skill available across all projects:

```bash
git clone https://github.com/poofdotnew/poof-skill.git ~/.claude/skills/poof
```

#### Option C: Install as a plugin

If you have a plugin marketplace set up, you can install it via Claude Code's `/plugin` command:

```
/plugin marketplace add poofdotnew/poof-skill
/plugin install poof@poof-skill
```

#### Using it

Once installed, just ask Claude Code to build a Poof project — the skill activates automatically on keywords like `poof project`, `deploy dApp`, `create poof app`, `@pooflabs/server`, etc.

Example prompts:

```
> Create a poof project for a token-gated voting app
> Deploy my poof dApp to production
> Build a Solana app with poof that has a bonding curve and leaderboard
```

Claude Code will handle authentication, project creation, chat-driven iteration, testing, and deployment — all without leaving your terminal.

### Other AI Agents (Cursor, Windsurf, etc.)

Point your agent's context/knowledge configuration at the [`SKILL.md`](SKILL.md) file or the [`docs/`](docs/) directory. The skill auto-triggers on the same keywords listed above.

### What the skill provides

- **Authentication setup** — Solana keypair generation and Cognito JWT flow
- **30 MCP tools** — Complete API for project lifecycle (create, chat, deploy, test, download)
- **Generation modes** — `full`, `policy`, `backend,policy`, `ui,policy`
- **Polling patterns** — Timeout + exponential backoff helpers
- **Testing workflows** — Lifecycle action tests with pass/fail evaluation
- **Deployment pipeline** — Draft → Preview → Production with security scanning
- **Troubleshooting** — Common errors and recovery patterns

## Generation Modes

| Mode | What Poof generates | You build |
|------|--------------------|----|
| `full` (default) | Everything: UI, backend, policies, deployment | Nothing — turnkey |
| `policy` | Database policies + typed SDK | Your own frontend + backend |
| `backend,policy` | Backend API routes + policies | Your own frontend |
| `ui,policy` | Frontend UI + policies | Your own backend |

## Documentation

| Doc | Covers |
|-----|--------|
| [**SKILL.md**](SKILL.md) | Main skill reference — auth, workflow, checklist, best practices |
| [**How Poof Works**](docs/how-poof-works.md) | Architecture, policy system, plugins, on-chain vs off-chain |
| [**Building & Chat**](docs/building-and-chat.md) | Project creation, chat workflow, polling, full code examples |
| [**API Reference**](docs/api-reference.md) | All 30 MCP tools with inputs/outputs |
| [**Backend-Only Mode**](docs/backend-only.md) | Custom frontend with Poof backend |
| [**Local Frontend Guide**](docs/local-frontend-guide.md) | SDK init, wallet auth, database access, React hooks |
| [**Database SDK**](docs/database-sdk.md) | Generated typed SDK, collections, read/write patterns |
| [**Deployment**](docs/deployment.md) | Environments, publishing, code downloads, custom domains |
| [**Testing**](docs/testing.md) | Lifecycle actions, test syntax, bootstrap scripts |
| [**Credits & Payments**](docs/credits-and-payments.md) | Credit system, paid features, USDC top-up |
| [**Troubleshooting**](docs/troubleshooting.md) | Common errors and recovery patterns |
| [**Cross-Compatibility**](docs/cross-compatibility.md) | curl, Python, OpenAI function calling format |

## Environments

| Environment | URL | Use case |
|-------------|-----|----------|
| Production | `https://poof.new` | Live agents and deployed apps |
| Staging | `https://staging.poof.new` | Testing against staging |
| Local | `http://localhost:3000` | Local development |

Set `POOF_ENV` in your `.env` to switch. Defaults to `production`.

## License

[MIT](LICENSE) — Copyright (c) 2026 Tarobase, Inc.
