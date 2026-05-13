# Agents

Poof's agent runtime gives your local backend stateful, multi-turn LLM workflows with built-in web + browser tools — and a clean invocation API (`runAgent` / `streamAgent` / `enqueueAgentRun`) callable from any of your routes.

**Use an agent when:**
- The flow is multi-turn with state per `(agent, sessionId)`.
- The model needs to call tools (web search, page fetch, structured extraction, browser automation).
- The model returns a schema-validated object (with retry on schema miss).
- The work might run >30s and you want webhook + SSE progress.

**Use plain `aiRun` instead when:** one model call → one response, no memory, no tools.

## File layout

```text
partyserver/
├── poof-agents.json                # list of agents (read by Poof at deploy)
├── .flue/agents/<kebab>.ts         # the agent's handler — create one per agent
├── src/lib/poof-flue-runtime.ts    # platform runtime (do not edit)
├── src/lib/poof-mcp-fanout.ts      # platform fanout routes (do not edit)
└── src/index.ts                    # wire-up: definePoofAgent + registerMcpFanoutRoutes
```

Naming chain (must match exactly): `.flue/agents/<kebab>.ts` → `definePoofAgent('<kebab>', ...)` → `export const <Pascal> = ...`.

## 1 — write the agent handler

```ts
// partyserver/.flue/agents/translate.ts
import type { FlueContext } from '@flue/sdk/client';
import * as v from 'valibot';

export const triggers = { webhook: true };

const resultSchema = v.object({ translation: v.string() });

export default async function ({ init, payload }: FlueContext) {
  const harness = await init({ model: 'anthropic/claude-sonnet-4-6' });
  const session = await harness.session();
  const p = payload as { text: string; language: string };
  const { data } = await session.prompt(
    `Translate "${p.text}" to ${p.language}. Reply with JSON { "translation": "..." }`,
    { schema: resultSchema },
  );
  return data;
}
```

Default model: `anthropic/claude-sonnet-4-6` unless the user named one. Cast `payload` — the framework types it `unknown`.

## 2 — register in `src/index.ts`

```ts
import {
  configurePoofAIProviders,
  configurePoofAgentsRuntime,
  definePoofAgent,
} from './lib/poof-flue-runtime.js';
import { registerMcpFanoutRoutes } from './lib/poof-mcp-fanout.js';
import translateHandler from '../.flue/agents/translate.js';

configurePoofAIProviders();                                          // ONCE, module-load, before registrations
export const Translate = definePoofAgent('translate', translateHandler);
configurePoofAgentsRuntime(['translate']);                           // list every agent name

// ...existing middleware...
registerMcpFanoutRoutes(app);                                        // REQUIRED — enables MCP + AI tool calls from agents
// ...registerRoutes(app)...
```

Don't mount `app.route('/', flue())`. The agent HTTP surface is platform-internal; tenants invoke via the helpers below.

## 3 — list in `poof-agents.json`

```json
{ "agents": [{ "name": "translate" }] }
```

Poof reads this at deploy and emits the agent's Durable Object binding + migration into `wrangler.toml`. The UI's **Agents** tab reads this file from the deployed artifact.

## 4 — invoke from a tenant route

Don't expose agents publicly — call them from your own auth-gated `/api/*` routes:

```ts
import { runAgent, streamAgent, enqueueAgentRun, getAgentRun, streamAgentRunEvents } from '../lib/poof-flue-runtime.js';

// Sync — blocks for the final result
app.post('/api/translate', async (c) => {
  await validatePoofAuth(c);
  const { text, language, sessionId } = await c.req.json();
  const result = await runAgent<{ translation: string }>(c.env, 'translate', {
    sessionId,
    payload: { text, language },
  });
  return sendSuccess(c, result);
});

// Streaming — return SSE Response directly
app.post('/api/translate/stream', async (c) => {
  await validatePoofAuth(c);
  const body = await c.req.json();
  return streamAgent(c.env, 'translate', { sessionId: body.sessionId, payload: body });
});

// Webhook — fire and forget, poll for result
app.post('/api/translate/start', async (c) => {
  await validatePoofAuth(c);
  const body = await c.req.json();
  const { runId } = await enqueueAgentRun(c.env, 'translate', {
    sessionId: body.sessionId,
    payload: body,
  });
  return sendSuccess(c, { runId });
});

app.get('/api/translate/status', async (c) => {
  await validatePoofAuth(c);
  const run = await getAgentRun(c.env, 'translate', c.req.query('sessionId')!, c.req.query('runId')!);
  return sendSuccess(c, run);
});
```

**Continuing a session:** same `sessionId` = same DO instance = preserved conversation/tool state. Fresh sessionId = fresh session.

## Default tool surface (every agent gets these for free)

```text
web_search        $0.01      Public web search → ranked snippets + answer (Tavily)
web_fetch         $0.003     URL → clean markdown (JS-rendered)
web_extract       $0.01      URL + prompt → structured JSON via AI
web_screenshot    $0.005     URL → PNG

browse_start      $0.05      Open stateful browser session (returns session_id)
browse_navigate   $0.02      Move session to a URL
browse_act        $0.03      Natural-language action ("click sign in", "fill email")
browse_observe    $0.02      List interactable elements on current page
browse_extract    $0.04      Current page → structured data via prompt
browse_screenshot $0.02      Current page → PNG
browse_end        free       Close session (always call this when done)
```

Prices are flat per call, billed to your project. Prefer `web_fetch` / `web_extract` for read-only research; reach for `browse_*` only when state across pages matters (login, multi-step forms).

The model picks which tool based on your prompt — you don't import or wire any of them. To opt out entirely: `init({ poofTools: false })`.

## Triggering pattern matrix

| Trigger | Tenant route uses | How it connects |
|---|---|---|
| User clicks, agent replies | `runAgent` | Frontend awaits the response |
| User clicks, live progress | `streamAgent` | Frontend uses `EventSource` |
| User clicks, long-running, polled | `enqueueAgentRun` + `getAgentRun` | Frontend polls `/api/.../status` |
| Many jobs, retries + DLQ | **Queue** — consumer calls `enqueueAgentRun` | See `docs/queues.md` |
| On a schedule | **Heartbeat** — handler calls `runAgent` | See `docs/heartbeats.md` |

## Long-running agents (one session, multiple triggers)

For agents that "chug along" over hours/days — a negotiator that contacts dealers and waits for replies, a research agent that polls sources, a workflow that resumes when external events arrive — pin a **stable `sessionId`** per workflow and invoke the same agent from multiple triggers. The DO holds conversation history; Tarobase (or another store) holds business state that arrived between turns.

Pattern shape:

```ts
// .flue/agents/<workflow>.ts — one handler, multiple triggers
export default async function ({ init, payload }: FlueContext) {
  const harness = await init({ model: 'anthropic/claude-sonnet-4-6' });
  const session = await harness.session();
  const p = payload as
    | { trigger: 'start'; ...inputs }
    | { trigger: 'poll' }
    | { trigger: 'user_update'; message: string }
    | { trigger: 'inbound_event'; ...event };
  // Read durable state (counterparties, prices, status) from Tarobase
  // so each trigger sees what other handlers have stored since the
  // last wake-up.
  const state = await get(`workflows/${ctx.id}`);
  const { data } = await session.prompt(routerPromptFor(p, state), { schema });
  return data;
}
```

Then wire each trigger to the same `sessionId`:

```ts
// Tenant route — start
app.post('/api/negotiations', async (c) => {
  await validatePoofAuth(c);
  const sessionId = `neg-${userId}-${Date.now()}`;
  await set(`workflows/${sessionId}`, { status: 'active', ... });
  const { runId } = await enqueueAgentRun(c.env, 'car-negotiator', {
    sessionId, payload: { trigger: 'start', ...inputs },
  });
  return sendSuccess(c, { sessionId, runId });
});

// Tenant route — user reacts
app.post('/api/negotiations/:id/update', async (c) => {
  await validatePoofAuth(c);
  const { id } = c.req.param();
  await enqueueAgentRun(c.env, 'car-negotiator', {
    sessionId: id, payload: { trigger: 'user_update', message: (await c.req.json()).message },
  });
  return sendSuccess(c, {});
});

// Heartbeat — periodic poll over every active workflow
export async function pollWorkflowsHandler({ env }) {
  const { negotiations } = await getMany([{ collection: 'negotiations', where: { status: 'active' } }]);
  for (const n of negotiations) {
    await enqueueAgentRun(env, 'car-negotiator', { sessionId: n.id, payload: { trigger: 'poll' } });
  }
}

// Queue consumer — inbound external events (email reply, webhook, ...)
export async function handleInboundQueue(batch, env) {
  for (const msg of batch.messages) {
    const { sessionId, ...event } = msg.body;
    await runAgent(env, 'car-negotiator', { sessionId, payload: { trigger: 'inbound_event', ...event } });
    msg.ack();
  }
}
```

Each call is one short LLM run that loads context, advances the workflow, persists results, and exits. The agent isn't "running" continuously — it's a series of stateful, short invocations against the same session. That matches the worker model (no long-lived processes) and bounds each wake-up's compute + cost.

**State split:**
- **DO (`session`)**: conversation history, tool-call cache, prompt context. Auto-managed by the harness.
- **Tarobase / your data store**: business state others can read independently (status, counterparties, contracts, prices). The agent reads at wake-up, writes before exit.

Don't try to keep state only in the DO if you also need to query it from `/api/*` routes — Tarobase is the join point.

**Auto-compaction:** Flue compacts the session automatically when the running context approaches the model's window (default: keep ~20K recent tokens verbatim, summarize everything older into one condensed turn, reserve ~16K for the response). One internal LLM call per compaction, billed through the same proxy. The agent keeps running — there's no hard wall. The trade-off is that verbatim quotes and big tool-result blobs from much older turns become summary-only, which is the second reason business facts (prices, contracts, dealer contacts) belong in Tarobase rather than relying on conversation history to preserve them. Re-load them at each wake-up.

## Security model — what's gated and what isn't

- The auto-mounted `/agents/<name>/<session>` HTTP path is **not exposed**. Agents are invoked only via DO binding from your own routes.
- `/__poof/mcp/*` and `/__poof/ai/*` are the platform fanout routes the agent runtime uses internally. They're gated by a per-script `POOF_FANOUT_TOKEN` secret that Poof sets at deploy. Anonymous requests return 401.
- LLM and MCP attribution is sealed by Cloudflare's dispatcher — cross-tenant billing is impossible.
- A malicious tenant who leaks their own token can have their own project burned. Same threat profile as any leaked tenant API key. Bounded by the per-project overuse cap.

## Local dev vs. deployed

Agents only run on the **deployed** worker. Local `wrangler dev` can compile the code, but DO bindings + the platform fanout token aren't present locally — invocations return 401. Iterate on deployed draft; smoke through `/api/<route>` end-to-end.

## Don'ts

- Don't mount `app.route('/', flue())`. The HTTP surface is internal.
- Don't bypass `registerMcpFanoutRoutes(app)` — agents will fail with CF error 1016.
- Don't call `aiRun` from inside an agent handler. The agent's `init({ model })` configures providers correctly; `aiRun` from a DO escapes to public DNS.
- Don't reuse a tool name reserved by the platform (`mcp__poof__*`). The runtime hard-rejects collisions at `init()` with an opt-out hint.
- Don't leave `browse_start` sessions open — always `browse_end` when the flow finishes. Sessions auto-expire after `keep_alive_minutes` (max 10) but cost browser minutes the whole time.
