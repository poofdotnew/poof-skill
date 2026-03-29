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

This repository contains the **Poof skill** — a documentation and reference package that teaches AI agents (Claude Code, Cursor, Windsurf, etc.) how to use the [Poof](https://poof.new) platform programmatically via the `poof` CLI.

Poof is a Backend-as-a-Service for Solana. It generates full-stack dApps (database policies, backend APIs, frontend UI, blockchain operations) from natural language — and this skill gives your AI agent the knowledge to drive that entire lifecycle without a browser.

> **For AI agents:** The skill definition lives in [`SKILL.md`](SKILL.md). Detailed docs are in [`docs/`](docs/).

## How It Works

```
Your Agent ──► poof CLI (auth + API + polling built in) ──► poof.new
```

Your agent runs `poof` CLI commands to create projects, iterate via chat, run tests, and deploy — all through simple shell commands.

## Quick Start

### 1. Install the CLI

```bash
brew install poofdotnew/tap/poof
```

Or see the [poof-cli repo](https://github.com/poofdotnew/poof-cli) for other install methods.

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
| [**Building & Chat**](docs/building-and-chat.md) | Project creation, chat workflow, full code examples |
| [**CLI Command Reference**](docs/api-reference.md) | All CLI commands with inputs/outputs |
| [**Backend-Only Mode**](docs/backend-only.md) | Custom frontend with Poof backend |
| [**Local Frontend Guide**](docs/local-frontend-guide.md) | SDK init, wallet auth, database access, React hooks |
| [**Database SDK**](docs/database-sdk.md) | Generated typed SDK, collections, read/write patterns |
| [**Deployment**](docs/deployment.md) | Environments, publishing, code downloads, custom domains |
| [**Testing**](docs/testing.md) | Lifecycle actions, test files, bootstrap scripts, UI functional tests, expression syntax, testing strategy by layer |
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
