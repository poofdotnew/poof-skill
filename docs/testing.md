# Testing

Poof uses **lifecycle actions** — declarative JSON files — to test database policies and bootstrap app data. No test code needed.

## Contents
- [Why This Matters for Agents](#why-this-matters-for-agents)
- [Two Modes](#two-modes)
- [Core Operations](#core-operations)
- [Key Patterns](#key-patterns)
- [Bootstrap Configuration](#bootstrap-configuration)
- [Rules](#rules)
- [Credit Cost](#credit-cost)
- [Checking Test Results Programmatically](#checking-test-results-programmatically)
- [Asking Poof to Generate Tests](#asking-poof-to-generate-tests)

## Why This Matters for Agents

When you ask the Poof AI to add features via `chat`, it generates policies (rules, hooks, fields). Lifecycle actions let you validate those policies work correctly before deploying. You can ask the Poof AI to generate these test files for you via chat.

## Two Modes

### 1. Test Files — Validate Policies

Test files run server-side with mock actors in ephemeral apps. They verify that policy rules allow and deny the right operations.

```json
{
  "version": "1",
  "name": "Test user message permissions",
  "actors": {
    "Alice": "TestAddr1...",
    "Bob": "TestAddr2..."
  },
  "steps": [
    { "op": "as", "who": "Alice" },
    { "op": "set", "path": "messages/m1", "data": { "message": "Hello" } },
    { "op": "expect", "expr": "get(/messages/m1).message == 'Hello'" },

    { "op": "as", "who": "Bob" },
    { "op": "set", "path": "messages/m1", "data": { "message": "Hacked" }, "shouldFail": true }
  ]
}
```

**Location:** `lifecycle-actions/test-[name].json`

### 2. Bootstrap Files — Initialize App Data

Bootstraps set up required state (configs, counters, default data). They run client-side in the user's browser or server-side for vault operations.

```json
{
  "version": "1",
  "kind": "bootstrap",
  "scope": "owner",
  "executionTarget": "client",
  "targetEnvironments": ["mock", "preview", "live"],
  "bootstrapType": "deployment-idempotent",
  "description": "Initialize app configuration",
  "steps": [
    { "op": "ensure", "expr": "get(/config/app) == null" },
    { "op": "set", "path": "config/app", "data": { "version": 1, "enabled": true } }
  ]
}
```

**Location:** `lifecycle-actions/bootstrap-[scope].json`

## Core Operations

| Op | Purpose | Mode |
|----|---------|------|
| `as` | Switch actor | Tests only |
| `set` | Create/update document | Both |
| `setMany` | Atomic multi-doc write | Both |
| `delete` / `deleteMany` | Delete documents | Both |
| `expect` | Assert conditions | Tests only |
| `ensure` | Conditional execution (idempotency) | Both |
| `mock` | Mock plugin calls (on-chain hooks) | Tests only |
| `fund` | Give SOL/tokens to test accounts | Tests only |
| `applyBootstrap` | Load another bootstrap file | Both |
| `setTime` / `advanceTime` | Control test clock | Tests only |

## Key Patterns

### Test Both Success and Failure
```json
{ "op": "set", "path": "posts/p1", "data": { "title": "My Post" } },
{ "op": "expect", "expr": "get(/posts/p1).title == 'My Post'" },

{ "op": "as", "who": "Bob" },
{ "op": "set", "path": "posts/p1", "data": { "title": "Hijacked" }, "shouldFail": true }
```

### Mock On-Chain Hooks in Tests
```json
{ "op": "mock", "call": "@TokenPlugin.transfer", "result": true }
```

Always mock plugin calls in tests — they run in ephemeral environments without real blockchain access.

### Make Bootstraps Idempotent
```json
{ "op": "ensure", "expr": "get(/config/app) == null" },
{ "op": "set", "path": "config/app", "data": { "version": 1 } }
```

Use `ensure` so bootstraps are safe to run multiple times.

### Fund Test Accounts
```json
{ "op": "fund", "account": "$Alice", "amount": 1.0 },
{ "op": "fund", "account": "$Alice", "amount": 100, "mint": "USDC_MINT" }
```

Place `fund` steps at the start, before data operations that require balances.

## Bootstrap Configuration

| Field | Values | Default |
|-------|--------|---------|
| `executionTarget` | `"client"` or `"server"` | — (always specify) |
| `targetEnvironments` | `["mock"]`, `["mock", "preview", "live"]`, etc. | `["preview"]` |
| `bootstrapType` | `"seed-data"`, `"deployment-idempotent"`, `"manual-execution"` | `"manual-execution"` |
| `clearFirst` | `true` / `false` | `false` (only works with `["mock"]`) |

## Rules

- All lifecycle action files must be `.json`
- Client-side bootstraps must NOT use `as` (no actor switching)
- Standalone bootstraps must NOT use mocks (real blockchain)
- `clearFirst` only works when `targetEnvironments` is exactly `["mock"]`
- Tests always run in fresh ephemeral environments

## Credit Cost

Generating and running tests goes through `chat`, which costs credits (~1 credit per message). A build + test + fix cycle typically costs 3-5 credits total. Always check `get_credits` before starting test generation to avoid running out mid-workflow.

## Checking Test Results Programmatically

Use `get_test_results` to evaluate test outcomes with structured data instead of parsing chat messages:

```typescript
const testResults = await mcpCall('tools/call', {
  name: 'get_test_results',
  arguments: { projectId },
});

// Check if all tests passed
const allPassed = testResults.summary.total > 0 && testResults.summary.failed === 0 && testResults.summary.errors === 0;

// Get details for failed tests
const failures = testResults.results.filter((r: any) => r.status === 'failed' || r.status === 'error');
for (const f of failures) {
  console.log(`${f.fileName}: ${f.lastError}`);
}
```

The response includes:
- `results` — array of individual test executions with `status`, `counts`, `lastError`, `duration`
- `summary` — aggregate counts: `total`, `passed`, `failed`, `errors`, `running`

This is more reliable than parsing `get_messages` output — it reads directly from the structured `lifecycleExecutions` database.

## Asking Poof to Generate Tests

You can ask the Poof AI via `chat` to generate test files:

> "Generate lifecycle action tests for the staking policy — test that users can stake, can't double-stake, and can unstake after the lock period."

The AI knows the policy schema and will generate appropriate test files.
