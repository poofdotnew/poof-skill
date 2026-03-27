# Testing

Poof uses **lifecycle actions** — declarative JSON files — to test database policies, validate backend logic, and verify UI behavior. No test code needed.

## Contents
- [Why This Matters for Agents](#why-this-matters-for-agents)
- [Three Test Modes](#three-test-modes)
- [Core Operations](#core-operations)
- [Expression Syntax Reference](#expression-syntax-reference)
- [Actor Variable Quoting](#actor-variable-quoting)
- [Key Patterns](#key-patterns)
- [UI Functional Tests](#ui-functional-tests)
- [Bootstrap Configuration](#bootstrap-configuration)
- [Testing Strategy by Layer](#testing-strategy-by-layer)
- [Testing Best Practices](#testing-best-practices)
- [Testing Troubleshooting](#testing-troubleshooting)
- [Rules](#rules)
- [Credit Cost](#credit-cost)
- [Checking Test Results Programmatically](#checking-test-results-programmatically)
- [Asking Poof to Generate Tests](#asking-poof-to-generate-tests)

## Why This Matters for Agents

When you ask the Poof AI to add features via `chat`, it generates policies (rules, hooks, fields). Lifecycle actions let you validate those policies work correctly before deploying. You can ask the Poof AI to generate these test files for you via chat.

## Three Test Modes

### 1. Test Files — Validate Policies & Backend Logic

Test files run server-side with mock actors in ephemeral apps. They verify that policy rules allow and deny the right operations, and that hooks/plugins behave correctly.

```json
{
  "version": "1",
  "name": "Test user message permissions",
  "actors": {
    "Alice": "TestAddr1...",
    "Bob": "TestAddr2..."
  },
  "policyConstants": {
    "messages": { "MAX_LEN": 14 }
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

### 3. UI Test Files — Browser-Based Functional Tests

UI test files use browser automation (Browserbase + Stagehand) to verify the full stack works end-to-end: UI renders correctly, forms submit, navigation works, and on-chain operations complete.

All UI tests run as a mock authenticated user with address: **`HKbZbRR7jWWR5VRN8KFjvTCHEzJQgameYxKQxh2gPoof`**. The test runner opens the app with `?mockAuth=true` — no real wallet needed.

```json
{
  "version": 1,
  "name": "test-ui-create-post",
  "type": "ui-test",
  "description": "Verify users can create and view posts",
  "steps": [
    {
      "act": "Click the 'Create Post' button in the header",
      "verify": {
        "extract": "Is the post creation form visible?",
        "schema": { "formVisible": "boolean" },
        "expect": { "formVisible": true }
      }
    },
    {
      "act": "Type 'My First Post' in the title input and 'Hello world' in the content area, then click Submit",
      "verify": {
        "extract": "The text of the success message or the title of the newly created post",
        "schema": { "title": "string" },
        "expect": { "title": "My First Post" }
      }
    }
  ]
}
```

**Location:** `lifecycle-actions/ui-test-[feature].json`

See the full [UI Functional Tests](#ui-functional-tests) section below for format details, patterns, and how to trigger them.

## Core Operations

| Op | Purpose | Mode |
|----|---------|------|
| `as` | Switch actor | Tests only |
| `set` | Create/update document | Both |
| `setMany` | Atomic multi-doc write | Both |
| `delete` / `deleteMany` | Delete documents | Both |
| `expect` | Assert conditions (3 forms) | Tests only |
| `ensure` | Conditional execution (idempotency) | Both |
| `mock` | Mock plugin calls (on-chain hooks) | Tests only |
| `fund` | Give SOL/tokens to test accounts | Tests only |
| `applyBootstrap` | Load another bootstrap file | Both |
| `setTime` / `advanceTime` | Control test clock | Tests only |

### Operation Details

**`as`** — Switch the current actor. All subsequent steps run as that actor until the next `as`. Does NOT have a `do` field — place operations as separate steps after it. **Tests only; never in client-side bootstraps.**

**`expect`** — Three forms:
```json
{ "op": "expect", "expr": "get(/messages/m1) != null" }
{ "op": "expect", "left": "get(/messages/m1).message", "right": "Hello" }
{ "op": "expect", "not": "get(/messages/m2) != null" }
```

**`set` with `shouldFail`** — Test that an operation is denied by policy:
```json
{ "op": "set", "path": "posts/p1", "data": { "title": "Hijacked" }, "shouldFail": true }
```

**`mock`** — Mock a plugin call. Field is `returns` (not `return`). Must be placed BEFORE the operation that triggers the hook:
```json
{ "op": "mock", "pluginCall": "@TokenPlugin.transfer", "returns": true }
```

**`fund`** — Airdrop SOL or tokens. Works on ephemeral/offchain test environments only:
```json
{ "op": "fund", "account": "$Alice", "amount": 1.0 }
{ "op": "fund", "account": "$Alice", "amount": 100, "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v" }
```

## Expression Syntax Reference

Lifecycle action expressions use a DSL for data access and logic. Used in `expect`, `ensure`, and policy rules.

### Built-in Variables

| Variable | Description |
|----------|-------------|
| `@user.address` | Current actor's wallet address |
| `@time.now` | Current logical time (Unix seconds) |
| `@constants.COLLECTION.CONSTANT` | Policy constants (e.g., `@constants.messages.MAX_LEN`) |
| `$ActorName` | Actor address placeholder (e.g., `$Alice`) |

### Data Access

| Function | Description |
|----------|-------------|
| `get(/collection/docId)` | Retrieve document (before operation) |
| `getAfter(/collection/docId)` | Retrieve document (after operation, for atomic updates) |
| `get(/path).field` | Access a specific field |
| `get(/path/$Alice)` | Path with actor variable |

### Operators

**Comparison:** `==`, `!=`, `>`, `<`, `>=`, `<=`

**Logical:** `&&`, `||`, `!`

**Arithmetic:** `+`, `-`, `*`, `//` (integer division), `**` (exponentiation), `%` (modulo/remainder), `/%` (ceiling division)

### Utility Functions

| Function | Description |
|----------|-------------|
| `@StringUtils.length(str)` | Get string length |
| `@MathUtils.max(a, b)` | Maximum of two values |
| `@MathUtils.min(a, b)` | Minimum of two values |

### Expression Examples

```json
{ "op": "expect", "expr": "get(/stakes/s1).amount > 0" }
{ "op": "expect", "expr": "@StringUtils.length(get(/posts/p1).title) <= @constants.posts.MAX_TITLE_LEN" }
{ "op": "expect", "expr": "get(/config/app).version == 2 && get(/config/app).enabled == true" }
{ "op": "ensure", "expr": "get(/config/app) == null", "then": [{ "op": "set", "path": "config/app", "data": { "version": 1 } }] }
```

## Actor Variable Quoting

Actor variables (`$Alice`, `$Bob`, etc.) are replaced with addresses BEFORE expression evaluation. This creates a critical quoting rule:

**In paths** — use `$Alice` directly (no quotes):
```json
{ "op": "set", "path": "items/$Alice", "data": { "owner": "$Alice" } }
```

**In string comparisons within expressions** — MUST quote with single quotes:
```json
{ "op": "expect", "expr": "get(/items/i1).owner == '$Alice'" }
```

Without quotes, the expression compares the field value against the raw address string without proper string delimiters, causing evaluation errors.

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
{ "op": "mock", "pluginCall": "@TokenPlugin.transfer", "returns": true }
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
{ "op": "fund", "account": "$Alice", "amount": 100, "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v" }
```

Place `fund` steps at the start, before data operations that require balances.

### Testing with Bootstrap
```json
{
  "version": "1",
  "name": "Test with setup data",
  "actors": { "Owner": "OwnerAddr..." },
  "steps": [
    { "op": "applyBootstrap", "file": "lifecycle-actions/bootstrap-base.json" },
    { "op": "as", "who": "Owner" },
    { "op": "expect", "expr": "get(/config/app) != null" }
  ]
}
```

Use `applyBootstrap` to load shared setup data before running test assertions.

### Policy Constants in Tests
```json
{
  "version": "1",
  "name": "Test with custom constants",
  "actors": { "Alice": "TestAddr1..." },
  "policyConstants": {
    "messages": { "MAX_LEN": 14 }
  },
  "steps": [
    { "op": "as", "who": "Alice" },
    { "op": "set", "path": "messages/m1", "data": { "message": "Hello World!!!" } },
    { "op": "expect", "expr": "@StringUtils.length(get(/messages/m1).message) <= @constants.messages.MAX_LEN" }
  ]
}
```

Override policy constants in tests to validate constraint logic without modifying the actual policy.

### Time-Dependent Testing
```json
{ "op": "setTime", "epoch": 1700000000 },
{ "op": "as", "who": "Alice" },
{ "op": "set", "path": "stakes/s1", "data": { "staker": "$Alice", "amount": 1000000000, "lockedUntil": 1700003600 } },

{ "op": "advanceTime", "seconds": 3601 },
{ "op": "set", "path": "unstakes/u1", "data": { "staker": "$Alice", "stakeId": "s1" } },
{ "op": "expect", "expr": "get(/unstakes/u1) != null" }
```

Use `setTime` and `advanceTime` to test lock periods, cooldowns, and time-gated features.

### Funding Accounts with Specific Tokens
```json
{
  "version": "1",
  "name": "Test token purchase flow",
  "actors": { "Buyer": "TestAddr..." },
  "steps": [
    { "op": "fund", "account": "$Buyer", "amount": 0.1 },
    { "op": "fund", "account": "$Buyer", "amount": 100, "mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v" },
    { "op": "mock", "pluginCall": "@TokenPlugin.transfer", "returns": true },
    { "op": "as", "who": "Buyer" },
    { "op": "set", "path": "purchases/p1", "data": { "buyer": "$Buyer", "amount": 50, "currency": "USDC" } },
    { "op": "expect", "expr": "get(/purchases/p1).buyer == '$Buyer'" }
  ]
}
```

Fund with native SOL (omit `mint`) and specific tokens (pass `mint` address).

## UI Functional Tests

UI test files verify that the app's frontend works correctly — buttons function, forms submit, data displays, navigation works. They use browser automation (Browserbase + Stagehand) with natural-language-driven interactions and structured assertions.

### How They Work

1. The Poof AI generates `ui-test-*.json` files when you ask via `chat`
2. The test runner opens your draft app in a real browser with mock authentication
3. Each step executes a natural language `act` instruction, then runs a structured `verify` assertion
4. Results are available via `get_test_results` alongside policy test results

External agents trigger UI tests the same way as policy tests: `chat` → `pollUntilDone` → `get_test_results`.

### Mock Test User

All UI tests run as: **`HKbZbRR7jWWR5VRN8KFjvTCHEzJQgameYxKQxh2gPoof`**

The test runner automatically:
1. Opens the app with `?mockAuth=true` (no real wallet needed)
2. Seeds `sessionStorage` with the mock address for SDK auto-login
3. Clicks "Connect Wallet" as a fallback if auto-login doesn't trigger

Policy rules using `$userId == @user.address` will match this address. Data written by the test user appears under this address.

### Funding the Mock User (Required for Onchain Features)

If the app has onchain features (staking, transfers, swaps), the mock user needs tokens in their Poofnet wallet before UI tests run. Ask the Poof AI to fund the test user:

> "Fund the mock test user HKbZbRR7jWWR5VRN8KFjvTCHEzJQgameYxKQxh2gPoof with 10 SOL and 100 USDC on Poofnet before running UI tests."

### File Format

```json
{
  "version": 1,
  "name": "test-ui-<feature>",
  "type": "ui-test",
  "description": "What this test verifies",
  "steps": [
    {
      "act": "Natural language instruction for what to do",
      "verify": {
        "extract": "Question about what to observe on the page",
        "schema": { "fieldName": "type" },
        "expect": { "fieldName": "expectedValue" }
      }
    }
  ]
}
```

### Schema Types

`"boolean"`, `"string"`, `"number"`, `"string[]"`, `"number[]"`

### Expect Matchers

- **Exact values:** `{ "fieldName": true }`, `{ "title": "My App" }`
- **Array contains:** `{ "items": { "contains": "Buy groceries" } }`
- **Number comparison:** `{ "count": { "gte": 1 } }`, `{ "count": { "lte": 10 } }`

### Writing Good `act` Instructions

- Be specific: `"Click the blue 'Add Task' button in the header"`
- Include exact text: `"Type 'Buy groceries' in the input labeled 'Task name'"`
- One action per step — avoid compound: "click X then type Y then click Z"
- Use the app's actual UI text, not generic descriptions
- The browser starts on the app's homepage with mock auth — do NOT include URLs
- The user is already logged in — do NOT include "Connect Wallet" steps

### Writing Good `verify` Blocks

- `extract`: Natural language description of what to observe on the page
- `schema`: Expected data type for extracted values
- `expect`: Expected value with matcher semantics
- Verify the RESULT of the action, not the action itself
- Extract concrete, observable data (not opinions)

### Common UI Test Patterns

**Form submission:**
```json
{
  "act": "Type 'Buy groceries' in the task name input and click 'Add Task'",
  "verify": {
    "extract": "The list of task names visible on the page",
    "schema": { "tasks": "string[]" },
    "expect": { "tasks": { "contains": "Buy groceries" } }
  }
}
```

**Navigation:**
```json
{
  "act": "Click the 'Settings' link in the navigation menu",
  "verify": {
    "extract": "The page heading text",
    "schema": { "heading": "string" },
    "expect": { "heading": "Settings" }
  }
}
```

**Toggle:**
```json
{
  "act": "Click the toggle switch next to 'Enable notifications'",
  "verify": {
    "extract": "Is the notifications toggle in the on position?",
    "schema": { "enabled": "boolean" },
    "expect": { "enabled": true }
  }
}
```

**Delete:**
```json
{
  "act": "Click the delete button on the first item in the list and confirm the deletion",
  "verify": {
    "extract": "The number of items remaining in the list",
    "schema": { "count": "number" },
    "expect": { "count": { "gte": 0 } }
  }
}
```

**Error handling:**
```json
{
  "act": "Click the 'Submit' button without filling in any fields",
  "verify": {
    "extract": "Is an error message visible on the page?",
    "schema": { "errorVisible": "boolean" },
    "expect": { "errorVisible": true }
  }
}
```

**Onchain operation (fund first):**
```json
{
  "act": "Enter '1' in the stake amount field and click 'Stake SOL'",
  "verify": {
    "extract": "The confirmation message or staked amount shown on the page",
    "schema": { "staked": "boolean" },
    "expect": { "staked": true }
  }
}
```

### Execution Workflow

1. **Check if app has onchain features** — look at the policy for `onchain: true` collections
2. **Fund the mock test user if needed** — ask the Poof AI via `chat` to fund the test user before running UI tests
3. **Ask the Poof AI to generate and run UI tests** — same `chat` → `pollUntilDone` pattern
4. **Check results** — `get_test_results` includes UI test results alongside policy test results
5. Tests run sequentially (they may modify shared app state)
6. Each test gets its own browser session
7. Screenshots are captured after every step for debugging

## Bootstrap Configuration

| Field | Values | Default |
|-------|--------|---------|
| `executionTarget` | `"client"` or `"server"` | — (always specify) |
| `targetEnvironments` | `["mock"]`, `["mock", "preview", "live"]`, etc. | `["preview"]` |
| `bootstrapType` | `"seed-data"`, `"deployment-idempotent"`, `"manual-execution"` | `"manual-execution"` |
| `clearFirst` | `true` / `false` | `false` (only works with `["mock"]`) |

### Bootstrap Types

1. **`"seed-data"`** — Mock/sample data for testing. Use with `targetEnvironments: ["mock"]`.
2. **`"deployment-idempotent"`** — Runs on every deployment, safe to re-run. Use with `["mock", "preview", "live"]`.
3. **`"manual-execution"`** — Only run manually by admin. Default if omitted.

### Target Environments

- **`"mock"`** — Mock/draft environment (Poofnet). Safe to clear and reset.
- **`"preview"`** — Staging/devnet. Default for client-side bootstraps.
- **`"live"`** — Production/mainnet.
- **`"test"`** — Ephemeral (server-side tests only). Fresh per test run.

### Execution Targets

1. **Client-side (`"client"`)**: Runs in browser with admin's authenticated wallet. Use for 90% of bootstraps.
2. **Server-side (`"server"`)**: Runs on backend with access to vault private keys and Cloudflare secrets. Use for vault operations.

### clearFirst

Set `clearFirst: true` to wipe all offchain data before executing bootstrap steps. Only works when `targetEnvironments` is exactly `["mock"]`.

```json
{
  "version": "1",
  "kind": "bootstrap",
  "scope": "seed-users",
  "executionTarget": "client",
  "targetEnvironments": ["mock"],
  "bootstrapType": "seed-data",
  "clearFirst": true,
  "description": "Clear and reseed sample users in mock environment",
  "steps": [
    { "op": "setMany", "writes": [
      { "path": "users/user1", "data": { "name": "Alice" } },
      { "path": "users/user2", "data": { "name": "Bob" } }
    ]}
  ]
}
```

## Testing Strategy by Layer

### Policy Tests (`test-*.json`)

Validate access control and data integrity rules:
- **CRUD permissions per role**: Test owner, admin, public, and unauthorized actors for each collection
- **Ownership validation**: Verify `@newData.owner == @user.address` checks work
- **Field constraints**: Test validation rules (max length, required fields, readonly after creation)
- **Cross-collection reads**: Use `get()` to verify state across collections
- **Denial cases**: Always test `shouldFail: true` for unauthorized operations

```
Example: test-user-permissions.json → Alice creates her profile, Bob can't edit it
Example: test-admin-actions.json → Admin can delete posts, regular users can't
```

### Backend Tests (`test-*.json` targeting hooks/plugins)

Validate data operations that trigger on-chain hooks:
- **Hook execution**: Test that writing to a collection with hooks processes correctly
- **Mock all plugins**: Always mock `@TokenPlugin`, `@DeFiPlugin`, `@BondingCurvePlugin`, etc.
- **Time-dependent logic**: Use `setTime`/`advanceTime` for lock periods, cooldowns, expiry
- **Fund actors first**: Use `fund` before operations that check or require balances

```
Example: test-staking-flow.json → Fund Alice, mock transfer, stake, verify state
Example: test-lock-period.json → Set time, stake, try early unstake (shouldFail), advance time, unstake
```

### UI Tests (`ui-test-*.json`)

Validate the full stack through browser interaction:
- **Form submission**: Create, edit, and submit forms
- **Navigation**: Verify routing between pages
- **CRUD through UI**: Create, read, update, delete via the interface
- **Error handling**: Submit invalid input, verify error messages
- **Onchain interactions**: Fund mock user first, then test staking/transfers/swaps

```
Example: ui-test-create-post.json → Fill form, submit, verify post appears in list
Example: ui-test-staking.json → Fund user, enter amount, stake, verify confirmation
```

### Recommended Testing Order

1. **Policy tests first** — fast, isolated, no browser needed
2. **Backend tests second** — hooks, plugins, time logic
3. **UI tests last** — slowest (requires built draft app + browser), validates full stack

## Testing Best Practices

### Split Tests Into Separate Files
- **ONE test file per logical concern** — don't create one huge file with all test cases
- Create files like `test-user-create.json`, `test-user-permissions.json`, `test-admin-actions.json`
- Aim for 5-15 steps per file; start with 1-2 files, expand incrementally
- Easier to debug, faster to run, more maintainable

### Use Meaningful Names
- Test files: `test-user-permissions.json`, `test-token-transfers.json`, `test-lock-period.json`
- Bootstrap files: `bootstrap-owner.json`, `bootstrap-seed-users.json`
- UI test files: `ui-test-create-post.json`, `ui-test-staking.json`
- Document IDs: Simple identifiers like `app`, `m1`, `s1` (not kebab-case like `gaming-mouse`)

### Organize Test Steps
1. **Setup**: `applyBootstrap`, `fund`, `setTime`, `mock`
2. **Action**: `set`, `setMany`, `delete`
3. **Validation**: `expect`

### Test Both Success AND Failure
For every permission boundary, test that authorized operations succeed AND unauthorized operations fail (`shouldFail: true`).

### Distinguish Test vs Production Bootstraps
- **Test bootstraps** (`executionTarget: "server"`): Loaded by tests via `applyBootstrap`, run on ephemeral apps, CAN use mocks
- **Production bootstraps** (`executionTarget: "client"`): Run in user's browser, execute real transactions, NEVER use mocks

### Mock All Plugin Calls in Tests
Always mock `@TokenPlugin.transfer`, `@BondingCurvePlugin.calculatePrice`, etc. in test files. Ephemeral test environments don't have real blockchain access.

### Fund Before Operations
Place `fund` steps at the beginning, before any data operations that require balances. Fund with native SOL (omit `mint`) and specific tokens (pass `mint`).

## Testing Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `ruleDenied` | Policy rule denied the operation | Check the actor has the right address, required fields are present, data meets validation constraints |
| Bootstrap not executing | Wrong `executionTarget` or `ensure` condition blocking | Verify `executionTarget` is set; check that `ensure` expressions aren't always false |
| Mock not working | Mock placed after the operation that triggers it | Place `mock` step BEFORE the `set` that triggers the hook |
| Fund operation failing | Invalid account or wrong environment | Ensure account is a valid actor variable (`$Alice`) or address; fund only works in test environments |
| Insufficient balance | Actor not funded before on-chain operation | Add `fund` step for the actor before the operation that requires a balance |
| Expression evaluation error | Syntax issue in `expect` or `ensure` | Check: actor variables quoted in strings (`'$Alice'`), operators correct (`==` not `===`, `//` not `/`), paths start with `/` |
| `as` in bootstrap | Used `as` in a client-side bootstrap | Remove `as` — client-side bootstraps run as the authenticated admin only |
| UI test timeout | `act` instruction too vague or element not found | Make instructions more specific; verify the draft app is built and accessible |
| UI test verify fails | Extracted value doesn't match expected | Check schema type matches actual data; loosen expect matcher if appropriate (use `gte` instead of exact) |

## Rules

- All lifecycle action files must be `.json`
- Client-side bootstraps must NOT use `as` (no actor switching in browser)
- Standalone bootstraps must NOT use mocks (they need real blockchain interactions)
- `clearFirst` only works when `targetEnvironments` is exactly `["mock"]`
- Tests always run in fresh ephemeral environments
- Mock field is `returns` (not `return`)

## Credit Cost

Generating and running tests goes through `chat`, which costs credits (~1 credit per message). A build + test + fix cycle typically costs 3-5 credits total. Always check `get_credits` before starting test generation to avoid running out mid-workflow.

## Checking Test Results Programmatically

Use `get_test_results` to evaluate test outcomes with structured data instead of parsing chat messages. This returns results for **both policy tests and UI tests**.

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

You can ask the Poof AI via `chat` to generate all three types of test files:

**Policy tests:**
> "Generate lifecycle action tests for the staking policy — test that users can stake, can't double-stake, and can unstake after the lock period."

**Backend tests with hooks:**
> "Generate lifecycle action tests for the token transfer hooks — mock the plugin calls and verify the transfer records are created correctly."

**UI functional tests:**
> "Generate UI functional tests for the app. Test creating a post, navigating to the details page, and deleting a post. Fund the mock test user if the app has onchain features."

The AI knows the policy schema and will generate appropriate test files for each layer.
