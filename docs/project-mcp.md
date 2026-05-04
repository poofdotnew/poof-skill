# Project MCP

Project MCP is the direct model-to-project interface for an existing Poof project. It lets an external AI agent inspect project state, query or mutate project data, run lifecycle/UI tests, trigger draft Heartbeat tasks, and request Poofnet faucet tokens without sending a chat turn through `poof iterate`.

Use Project MCP when you need facts or direct project operations. Use `poof iterate` when you need Poof's hosted AI to edit code, generate implementation changes, or walk a build/deploy task. Use Platform MCP when one MCP client must manage many projects from one configured server.

## MCP Surfaces

Poof currently has two HTTP MCP surfaces with different ergonomics:

- **Platform MCP**: `/api/mcp`. One MCP server for the account/platform. Tools take `projectId` as an argument and cover lifecycle/admin operations such as `create_project`, `chat`, `publish_project`, `deploy_static_frontend`, `download_code`, `set_secrets`, domains, credits, and AI preferences.
- **Project MCP**: `/api/project/<project-id>/mcp`. One MCP server scoped to a single project. Tools do not take `projectId`; the URL selects the project. This endpoint is optimized for project inspection, database/data-plane work, lifecycle/UI test execution, draft Heartbeat trigger, and Poofnet faucet without a hosted AI chat turn.

For a client that manages many projects from one MCP config, use Platform MCP or a future project-hub MCP with explicit `projectId` arguments. For a client focused on one app/repo, Project MCP is safer and easier for the model because the project cannot be accidentally switched mid-tool-call.

## Interface Selection

| Interface | MCP/server shape | Scope | Supported here | Main overlap |
| --- | --- | --- | --- | --- |
| CLI | Shell commands | Current agent session | Full lifecycle, auth refresh, long polling, signing/payment helpers, archive upload/download, data-plane access | Most lifecycle/status/data tasks are also available through one MCP surface. |
| Platform MCP | One server: `/api/mcp` | Account/platform; tools take `projectId` | Project create/list/delete, chat, AI status, messages, project status, tasks, test results, publish/deploy, static deploy, source download, image upload, secrets, domains, credits, templates, logs, AI prefs | Overlaps CLI strongly, but is usable by MCP clients that cannot run shell commands. |
| Project MCP | One server per project: `/api/project/<project-id>/mcp` | Single project selected by URL | Project info, recent tasks/messages, policy/schema/constants, secret requirement names, logs, backend actions/URLs, file tree, lifecycle files/executions, project credit/blocked status, database/data-plane reads and draft writes, storage file URLs, lifecycle/UI test execution, draft heartbeat trigger, Poofnet faucet | Overlaps CLI `project status`, `logs`, `task`, `data`, and some verification flows, but avoids shell parsing and prevents accidental project switches. |
| `poof data --app-id` | CLI data-plane command | Tarobase appId | AppId/shared-library data access without Poof project ownership | Overlaps Project MCP database tools only when you have a Poof project and want its environment appId resolved for you. |

The important client setup detail: Project MCP is intentionally **one MCP server per Poof project**. A normal MCP client should register one URL per project it wants to expose. If that is not acceptable for the client, use Platform MCP and pass `projectId` to each tool instead.

Project MCP URLs use the Poof **project ID**, not a Tarobase appId. AppIds still matter for data-plane routing behind the scenes and for `poof data --app-id`, but project-level tools need the project UUID so they can resolve tasks, messages, policy files, deployments, and environment-specific appIds.

## Endpoint

```text
Production: https://poof.new/api/project/<project-id>/mcp
Staging:    https://v2-staging.poof.new/api/project/<project-id>/mcp
Local:      http://localhost:3000/api/project/<project-id>/mcp
```

`<project-id>` must be the Poof project UUID, not a Tarobase appId.

`GET` returns discovery metadata, tool policy, and tool schemas. `POST` accepts MCP JSON-RPC 2.0 requests.

## Auth

Run normal CLI auth first:

```bash
poof auth login
poof auth login --env staging
poof auth status --json
```

Direct MCP calls use the cached Tarobase ID token:

```bash
PROJECT_ID="<project-id>"
TOKEN="$(jq -r '.id_token' ~/.poof/tokens.json)"
MCP_URL="https://poof.new/api/project/$PROJECT_ID/mcp"
```

Send the token as either:

```text
Authorization: Bearer <token>
X-Tarobase-Token: <token>
```

The token wallet must be the project owner or a collaborator. The token environment must match the host: production token for `poof.new`, staging token for `v2-staging.poof.new`.

Never paste tokens into prompts, source files, docs, or issue comments. Keep them in shell variables or your MCP client's secret store.

## Staging Protection

`https://v2-staging.poof.new` may be behind Vercel deployment protection. If a request returns an HTML "Authentication Required" page, Poof MCP did not run. Use a Vercel-authenticated request path or a configured Vercel bypass secret for staging:

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  -H "x-vercel-protection-bypass: $VERCEL_BYPASS_TOKEN" \
  "$MCP_URL"
```

For browser-based follow-up requests, Vercel also supports setting a bypass cookie by adding `x-vercel-set-bypass-cookie: true` with the bypass header. Do not point external agents at internal dev-server/Fly hosts as a substitute for the public Project MCP URL; those are service-to-service paths with separate API-key requirements.

## JSON-RPC Methods

Project MCP supports:

- `initialize`
- `notifications/initialized`
- `ping`
- `tools/list`
- `tools/call`

Batches are supported up to 50 JSON-RPC messages.

## Examples

Discover the project MCP server:

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  "$MCP_URL"
```

List tools:

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' \
  "$MCP_URL"
```

Read project info:

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_project_info","arguments":{}}}' \
  "$MCP_URL"
```

Read recent tasks:

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"get_recent_tasks","arguments":{"limit":5}}}' \
  "$MCP_URL"
```

Read draft database data:

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"get","arguments":{"path":"<collection-or-document-path>","environment":"draft"}}}' \
  "$MCP_URL"
```

Run one uploaded lifecycle action:

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"execute_lifecycle_action","arguments":{"fileName":"test-example.json"}}}' \
  "$MCP_URL"
```

Run uploaded UI test files:

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":6,"method":"tools/call","params":{"name":"run_all_ui_tests","arguments":{"filePattern":"ui-test-*.json"}}}' \
  "$MCP_URL"
```

Trigger a draft Heartbeat task:

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":7,"method":"tools/call","params":{"name":"trigger_task","arguments":{"taskName":"weekly-publish","environment":"development"}}}' \
  "$MCP_URL"
```

Airdrop Poofnet test tokens:

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":8,"method":"tools/call","params":{"name":"request_faucet_tokens","arguments":{"walletAddress":"<wallet-address>","amount":1}}}' \
  "$MCP_URL"
```

## Exposed Tool Groups

Project MCP deliberately exposes a bounded project-scoped set:

- Project data: `get_current_server_time`, `get_project_info`, `get_recent_tasks`, `get_collections_list`, `get_current_policy`, `get_collection_schema`, `get_secrets_list`, `get_constants`, `get_server_logs`, `get_backend_actions`, `get_recent_messages`, `get_files_tree`, `get_lifecycle_action_executions`, `get_lifecycle_action_files`, `get_browser_use_executions`, `get_backend_urls`, `get_ai_model_config`, `get_project_blocked_status`, `get_project_credit_bank`
- Database/data-plane: `get`, `get_many`, `set`, `set_many`, `delete`, `delete_many`, `run_query_expression`, `generate_solana_addresses`, `set_file`, `get_files`
- Lifecycle and UI tests: `execute_lifecycle_action`, `run_all_lifecycle_tests`, `run_all_ui_tests`
- Draft operations: `trigger_task` for `environment="development"` only, `request_faucet_tokens` for Poofnet test tokens

Project MCP intentionally does not expose every Poof operation. Lifecycle/admin operations such as build, deploy, static deploy, source downloads, secret value writes, domains, credits, and project deletion are available through the CLI and/or Platform MCP, where the tool call carries an explicit `projectId` and uses the existing lifecycle checks. Hosted-AI-internal tools such as package management, generic browser automation, inline UI-test generation, rewind, memory, and mainnet heartbeat trigger assume Poof's managed AI workspace or internal permissions and should only be exposed externally after designing a dedicated, user-safe API for them.

## What Project MCP Adds

Compared with the CLI, Project MCP's main value is not exclusive business logic. It is a safer and more structured model interface:

- Tool schemas and descriptions are returned by `tools/list`; the model does not need to parse CLI help or free-form output.
- The project is fixed by the server URL, so a tool call cannot accidentally operate on a similarly named or stale project ID.
- Policy/schema, constants, backend action specs, recent messages, lifecycle files, lifecycle execution history, and file tree are available as direct tool calls.
- Database reads/writes resolve the correct draft/preview/live appId from the project, while still requiring an explicit `environment`.
- Lifecycle and UI tests can be executed directly against uploaded `lifecycle-actions/` files without asking Poof's hosted AI to run a chat turn.
- Draft Heartbeat tasks and Poofnet test-token faucet requests can be run directly for setup/debug flows that do not require code edits.

## Agent Decision Rules

- Prefer Project MCP for read-only diagnosis, project status, policy/schema discovery, constants, recent messages, logs, file tree lookup, database reads/writes, running already-authored lifecycle/UI tests, triggering draft Heartbeat tasks, and requesting Poofnet test tokens.
- Prefer `poof project status`, `poof task test-results`, and other CLI commands when the CLI already provides a stable task/project lifecycle command and you need CLI formatting or built-in polling.
- Prefer `poof iterate` when the desired output is new or modified project code, generated tests, design changes, or any task that needs Poof's hosted AI/code workspace.
- Do not treat MCP database writes as a replacement for policy-aware app behavior. They still run under the authenticated wallet and project policy; if policy rejects a write, inspect the policy/schema through MCP and fix the app or test setup intentionally.
