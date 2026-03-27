# API Reference

## Contents
- [MCP Endpoint](#mcp-endpoint)
- [All MCP Tools](#all-mcp-tools)
- [REST API Endpoints](#rest-api-endpoints)
- [Response Shapes](#response-shapes)
- [Error Handling](#error-handling)

## MCP Endpoint

**URL:** `POST ${POOF_BASE_URL}/api/mcp` (default: `https://poof.new/api/mcp`)

Implements MCP over HTTP using JSON-RPC 2.0.

### Discovery (No Auth)

```bash
curl https://poof.new/api/mcp  # or your staging URL
```

Returns server info, protocol version, capabilities, and tool names with descriptions. For full tool schemas (including `inputSchema`), use the `tools/list` JSON-RPC method via POST.

### JSON-RPC Methods

| Method | Purpose |
|--------|---------|
| `initialize` | Handshake — returns server info and capabilities |
| `tools/list` | List all available tools with input schemas |
| `tools/call` | Execute a tool by name with arguments |
| `notifications/initialized` | Client notification (no-op) |
| `ping` | Health check |

### Request Format

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "list_projects",
    "arguments": { "limit": 10 }
  }
}
```

### Response Format

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{ "type": "text", "text": "{\"projects\": [...]}" }]
  }
}
```

## All MCP Tools

### Project Management

| Tool | Description |
|------|-------------|
| `list_projects` | List all projects. Supports `limit`, `offset`. |
| `create_project` | Create project. AI builds from `firstMessage`. Use `generationMode` to scope output (e.g. `"policy"`, `"ui,policy"`). Requires `tarobaseToken`. |
| `update_project` | Update title, description, slug, visibility, generation mode. |
| `delete_project` | Delete a project and all its data. Irreversible. |

### AI Chat

| Tool | Description |
|------|-------------|
| `chat` | Send a message. AI builds features — backend, policy, UI, everything. Main interaction point. |
| `check_ai_active` | Check if AI is processing. Poll after `chat`. |
| `get_messages` | Get conversation history (chronological, non-internal). |

### Execution Control

| Tool | Description |
|------|-------------|
| `cancel_ai` | Cancel an in-progress AI execution. Use when the AI is stuck or you want to abort. |
| `steer_ai` | Redirect the AI mid-execution without cancelling. Injects a steering message to course-correct. |

### File Access

| Tool | Description |
|------|-------------|
| `get_files` | Get all source files for a project with contents. **Requires a credit purchase.** |
| `update_files` | Directly modify project source files. Pass a map of file paths to contents. |

### Status & Tasks

| Tool | Description |
|------|-------------|
| `get_project_status` | Metadata, task status, publish state, deployment URLs. |
| `list_tasks` | List tasks (builds, deployments, downloads). |
| `get_task` | Task details — poll for deployment/download progress. |
| `get_test_results` | Structured lifecycle test results — pass/fail, error messages, counts. Use instead of parsing chat messages. |

### Deployment & Publishing

| Tool | Description |
|------|-------------|
| `check_publish_eligibility` | Pre-deploy check: payment status, security review, readiness. |
| `publish_project` | Deploy. Targets: `preview`, `production`, `mobile`. Requires a credit purchase. |
| `deploy_static_frontend` | Deploy a pre-built static frontend (tar.gz, base64-encoded). See [static-deploy.md](static-deploy.md). |
| `download_code` | Start code export. Returns `taskId` — poll then `get_download_url`. The zip includes generated `db-client` + `collections/` SDK files. |
| `get_download_url` | Signed S3 URL for completed download (5-minute expiry). |

### Credits & Payments

| Tool | Description |
|------|-------------|
| `get_credits` | Credit balance: daily, add-on, totals. |
| `topup_credits` | Buy credits via x402 USDC. Also unlocks paid features (deployment, downloads). See [credits-and-payments.md](credits-and-payments.md). |

### Security

| Tool | Description |
|------|-------------|
| `security_scan` | Run a security audit. Analyzes policies, code, and config for vulnerabilities. |

### Templates

| Tool | Description |
|------|-------------|
| `list_templates` | Browse available project templates. Filter by `category`, `search`, `sortBy`. |

### Secrets

| Tool | Description |
|------|-------------|
| `get_secrets` | Get secret names and requirements per environment (not values). |
| `set_secrets` | Store secret values (API keys, tokens) for a project. Encrypted at rest. |

### Custom Domains

| Tool | Description |
|------|-------------|
| `get_domains` | List custom domains configured for a project. **Requires a credit purchase.** |
| `add_domain` | Add a custom domain. Project must be deployed to production first. **Requires a credit purchase.** |

### Logs

| Tool | Description |
|------|-------------|
| `get_logs` | Get runtime logs for a deployed project. Supports `environment` and `limit` params. |

### AI Preferences

| Tool | Description |
|------|-------------|
| `get_ai_preferences` | Model tier per use case. |
| `set_ai_preferences` | Update tiers: `average`, `smart`, `genius`. Requires a credit purchase. |

## REST API Endpoints

All MCP tools map to REST endpoints. Same auth headers required.

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/project` | List projects |
| POST | `/api/project/[id]` | Create project |
| PUT | `/api/project/[id]` | Update project |
| DELETE | `/api/project/[id]` | Delete project |
| POST | `/api/project/[id]/chat` | Send chat message |
| GET | `/api/project/[id]/ai/active` | Check AI status |
| GET | `/api/project/[id]/messages` | Get messages |
| GET | `/api/project/[id]/status` | Get project status |
| GET | `/api/project/[id]/tasks` | List tasks |
| POST | `/api/project/[id]/deploy-mainnet-preview` | Deploy preview |
| POST | `/api/project/[id]/deploy-prod` | Deploy production |
| POST | `/api/project/[id]/deploy-static` | Deploy a pre-built static frontend (`tar.gz` upload). See [static-deploy.md](static-deploy.md). |
| POST | `/api/project/[id]/mobile/publish` | Publish mobile app |
| POST | `/api/project/[id]/cancel` | Cancel AI execution |
| POST | `/api/project/[id]/steer` | Steer AI mid-execution |
| GET | `/api/project/[id]/files` | Get project files |
| POST | `/api/project/[id]/files/update` | Update project files |
| POST | `/api/project/[id]/security-scan` | Run security scan |
| GET | `/api/project/[id]/secrets` | Get secret names |
| POST | `/api/project/[id]/secrets` | Set secret values |
| GET | `/api/project/[id]/domains` | List custom domains |
| POST | `/api/project/[id]/domains` | Add custom domain |
| GET | `/api/project/[id]/logs` | Get runtime logs |
| POST | `/api/project/[id]/download` | Start code export |
| POST | `/api/project/[id]/download/get-signed-url` | Get signed download URL |
| GET | `/api/project/[id]/test-results` | Get structured test results |
| GET | `/api/project/[id]/task/[taskId]` | Get task details |
| GET | `/api/project/[id]/check-publish-eligibility` | Check publish eligibility |
| GET | `/api/template` | List templates |
| GET | `/api/user/credits` | Get credit balance |
| GET | `/api/user/ai-preferences` | Get AI preferences |
| PUT | `/api/user/ai-preferences` | Set AI preferences |
| POST | `/api/credits/topup` | x402 credit purchase |

## Response Shapes

Each tool returns its data inside `result.content[0].text` as a JSON string. Here are the parsed shapes for commonly used tools:

| Tool | Parsed Response Shape |
|------|----------------------|
| `get_credits` | `{ credits: { daily: { remaining, allotted, resetsAt }, addOn: { remaining, purchased }, total } }` |
| `get_messages` | `{ messages: [{ id, role: "user" \| "assistant", content: string, createdAt, status }] }` — **not a bare array**. `status` is `"queued"`, `"processing"`, or `"completed"` |
| `check_ai_active` | `{ active: boolean, status: "ok" \| "error" }` — `status` is `"error"` if the AI server is unreachable (vs `"ok"` when actively responding) |
| `get_project_status` | `{ project: { id, title, description, slug, ... }, latestTask: { id, status, title, ... }, publishState: { draft, preview, live, mobile }, urls: { preview, production, draft, mainnetPreview }, connectionInfo: { draft: { tarobaseAppId, backendUrl } \| null, preview: { tarobaseAppId, backendUrl } \| null, production: { tarobaseAppId, backendUrl } \| null, wsUrl, apiUrl, authApiUrl } }` |
| `create_project` | `{ success: true, projectId, message }` |
| `chat` | `{ success: true, messageId, queued: boolean }` — `queued` is `true` if AI was already active (HTTP 202). Messages execute in FIFO order |
| `list_projects` | `{ projects: [...] }` |
| `get_files` | `{ files: { [path: string]: string } }` — requires a credit purchase (returns `{ error, membershipRequired: true }` if unpaid) |
| `get_secrets` | `{ secrets: { required: string[], optional: string[] } }` |
| `get_domains` | `{ domains: [{ domain, isDefault, status }] }` — requires a credit purchase (returns `{ error, membershipRequired: true }` if unpaid) |
| `get_test_results` | `{ results: [{ id, fileName, testName, status, counts: { steps, expects, failed }, lastError, duration, startedAt }], summary: { total, passed, failed, errors, running } }` |
| `deploy_static_frontend` | `{ projectId, taskId, bundleUrl, slug }` |
| `get_logs` | `{ logs: [{ timestamp, level, message }] }` |
| `list_templates` | `{ templates: [{ id, name, slug, description, category }] }` |

> **Important:** Some responses may be plain strings rather than JSON objects. Always use try/catch when parsing `content[0].text`. See the robust `mcpCall` helper in [building-and-chat.md](building-and-chat.md).

## Error Handling

Errors come in the result payload:

```json
{
  "result": {
    "content": [{ "type": "text", "text": "{\"error\": \"Not authenticated\"}" }],
    "isError": true
  }
}
```

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Not authenticated` | Invalid/expired JWT | Call `getIdToken()` again |
| `Project not found` | Wrong project ID or not owner | Check project ownership |
| `You have run out of credits` | No credits remaining | Top up via x402 `topup_credits` |
| `Invalid projectId format` | Non-UUID project ID | Use `uuid.v4()` |

### Tips

- JWTs expire after ~1 hour — refresh with `getIdToken()` before each batch
- All IDs (projectId, messageId, taskId) must be UUIDs
- `get_credits` is free — check before starting long workflows
