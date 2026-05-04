<p align="center">
  <img src="https://poof.new/_next/image?url=%2Fimages%2Flogo-transparent-bg.png&w=256&q=75" alt="Poof" width="120" />
</p>

<h3 align="center">Poof CLI Skill</h3>

<p align="center">
  Build autonomous AI agents that create, deploy, and manage full-stack Solana dApps — using the <code>poof</code> CLI, no custom code needed.
</p>

<p align="center">
  <a href="https://poof.new">Website</a> ·
  <a href="#quick-start">Quick Start</a> ·
  <a href="docs/api-reference.md">CLI Command Reference</a> ·
  <a href="docs/troubleshooting.md">Troubleshooting</a>
</p>

---

## What is this?

This repository contains the **Poof skill** — a documentation and reference package that teaches AI agents (Claude Code, Cursor, Windsurf, etc.) how to use the [Poof](https://poof.new) platform programmatically via the `poof` CLI, Platform MCP, and project-scoped Poof MCP endpoint.

Poof is a Backend-as-a-Service for Solana. It generates full-stack dApps (database policies, backend APIs, frontend UI, blockchain operations) from natural language — and this skill gives your AI agent the knowledge to drive that entire lifecycle without a browser.

> **For AI agents:** The skill definition lives in [`SKILL.md`](SKILL.md). Detailed docs are in [`docs/`](docs/).

## How It Works

```
Your Agent ──► poof CLI (auth + API + polling built in) ──► poof.new
Your Agent ──► Platform MCP (/api/mcp, projectId args) ───► Poof account/projects
Your Agent ──► Project MCP (direct JSON-RPC tools) ────────► existing Poof project
```

Your agent runs `poof` CLI commands to create projects, iterate via chat, run tests, and deploy. For existing projects, it can also call Project MCP directly to read project info, inspect policy/schema/logs/messages/files, query or write data, run lifecycle/UI tests, trigger draft Heartbeat tasks, and request Poofnet faucet tokens without starting a `poof iterate` chat turn.

## Choose the Right Interface

Poof intentionally has overlapping surfaces. Pick based on scope and task:

| Interface | Scope | Best for | Avoid for |
|-----------|-------|----------|-----------|
| CLI | Current shell/session | Build/iterate/verify/ship, deploys, file uploads/downloads, payments/signing, long-running polling | Normal MCP clients that cannot run shell commands |
| Platform MCP `/api/mcp` | Account-level; every tool takes `projectId` when needed | One MCP client managing many projects; lifecycle/admin operations such as create, chat, deploy, static deploy, secrets, credits, domains | Single-project tools where accidental project switches are risky |
| Project MCP `/api/project/<project-id>/mcp` | One project selected by URL | Direct project inspection, policy/schema/constants/logs/messages/file tree, database get/set, storage file URLs, lifecycle/UI test execution, draft Heartbeat trigger, Poofnet faucet | Building/editing through Poof AI, deploys, secret value writes, credits, domains, mainnet heartbeat |
| `poof data --app-id` | One Tarobase appId | Shared primitives or appId-level data plane access outside a project workflow | Project lifecycle, tasks, messages, deploy state |

For a normal MCP client, Project MCP must be added once per Poof project because the project ID is part of the server URL. Use Platform MCP when a single MCP connection should manage many Poof projects.

## Quick Start

### 1. Install the CLI

```bash
brew install poofdotnew/tap/poof
```

Or see the [poof-cli repo](https://github.com/poofdotnew/poof-cli) for other install methods.
To update later, use `brew upgrade poofdotnew/tap/poof` for Homebrew installs, or `poof update` for self-managed binaries.

### 2. Set up credentials

```bash
poof keygen >> .env    # Generate a Solana keypair
poof auth login         # Authenticate
```

Or if you have existing keys, create a `.env`:

```env
SOLANA_PRIVATE_KEY=<your-base58-private-key>
SOLANA_WALLET_ADDRESS=<your-solana-public-key>
```

### 3. Create your first project

```bash
poof build -m "Build a token-gated voting app where holders can create proposals and vote"
```

This creates the project, waits for the AI to finish building, and returns the project ID and URLs.

### 4. Iterate

```bash
poof iterate -p <project-id> -m "Add a leaderboard page showing top voters"
```

### 5. Deploy

```bash
poof ship -p <project-id>
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

Once installed, just ask Claude Code to build a Poof project — the skill activates automatically on keywords like `poof project`, `deploy dApp`, `create poof app`, `poof CLI`, `poof build`, etc.

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

- **CLI commands** — Complete interface for project lifecycle (build, iterate, deploy, test, download)
- **Generation modes** — `full`, `policy`, `backend,policy`, `ui,policy`
- **Composite workflows** — `poof build` (create + wait), `poof iterate` (chat + wait + test results), `poof ship` (scan + check + deploy)
- **Platform MCP and Project MCP** — Account-scoped lifecycle/admin tools plus per-project direct inspection, database operations, lifecycle/UI tests, draft Heartbeat trigger, and Poofnet faucet
- **Testing workflows** — Lifecycle action tests, including source-authored UI tests for statically deployed local frontends
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
| [**Building & Chat**](docs/building-and-chat.md) | Project creation, chat workflow, full code examples |
| [**CLI Command Reference**](docs/api-reference.md) | All CLI commands with inputs/outputs |
| [**Backend-Only Mode**](docs/backend-only.md) | Custom frontend with Poof backend |
| [**Local Frontend Guide**](docs/local-frontend-guide.md) | SDK init, wallet auth, database access, React hooks |
| [**Database SDK**](docs/database-sdk.md) | Generated typed SDK, collections, read/write patterns |
| [**Deployment**](docs/deployment.md) | Environments, publishing, code downloads, custom domains |
| [**Testing**](docs/testing.md) | Lifecycle actions, test files, bootstrap scripts, UI functional tests, static-deploy UI test workflow, expression syntax, testing strategy by layer |
| [**Project MCP**](docs/project-mcp.md) | Direct model-to-project HTTP MCP endpoint, auth, exposed tools, examples, staging caveats |
| [**Credits & Payments**](docs/credits-and-payments.md) | Credit system, paid features, USDC top-up |
| [**Troubleshooting**](docs/troubleshooting.md) | Common errors and recovery patterns |

## Environments

| Environment | URL | Use case |
|-------------|-----|----------|
| Production | `https://poof.new` | Live agents and deployed apps |
| Staging | `https://v2-staging.poof.new` | Testing against staging |
| Local | `http://localhost:3000` | Local development |

Switch environments with the `--env` flag:

```bash
poof --env staging build -m "Test my app on staging"
```

## License

[MIT](LICENSE) — Copyright (c) 2026 Tarobase, Inc.
