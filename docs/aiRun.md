# Poof-Native AI: `aiRun`

`aiRun()` is the only LLM call shape supported by Poof's local backend. It routes through Poof's AI proxy on Cloudflare AI Gateway and meters usage against project credits (`$15 = 50 credits`, exact CF cost × 1.05). No tenant-held provider keys; Poof attaches sealed script identity at the edge.

Use `aiRun` for every chat completion. Do **not** import `openai` / `@anthropic-ai/sdk` / `@ai-sdk/*`, bind `AI` directly in `wrangler.toml`, or call provider URLs.

## Import

```ts
import { aiRun, AiBlockedError } from '../lib/poof-ai.js';
```

## Basic call (defaults work for 90% of cases)

```ts
const result = await aiRun(c.env, 'anthropic/claude-sonnet-4-6', {
  messages: [{ role: 'user', content: userInput }],
});
const text = result.choices[0].message.content;
```

Returns the OpenAI Chat Completions shape regardless of provider — `.choices[0].message.content` for text, `.choices[0].message.tool_calls` for tool calls.

## Picking a model

| Need | Model |
|---|---|
| General-purpose default | `anthropic/claude-sonnet-4-6` |
| Cheap / fast | `anthropic/claude-haiku-4-5` |
| Hardest reasoning | `anthropic/claude-opus-4-7` |
| OpenAI flagship | `openai/gpt-5.2` |
| Cheap Google | `google-ai-studio/gemini-2.5-flash` |
| CF-hosted Llama | `@cf/meta/llama-3.3-70b-instruct-fp8-fast` |

If a model name isn't recognized, trust the user and pass it through — the gateway returns a clear 404 if it doesn't exist.

## Cost + streaming

```ts
// Non-streaming with exact cost back to your app:
const { result, usage } = await aiRun(c.env, 'openai/gpt-4o-mini', { messages }, {
  includeUsage: true,
});
console.log(usage.costCredits);

// Streaming:
const stream = await aiRun(c.env, 'openai/gpt-4o-mini', { messages }, {
  stream: true,
  includeUsage: true,
});
return new Response(stream, { headers: { 'content-type': 'text/event-stream' } });
// With includeUsage the stream ends with:
//   event: poof-ai-usage
//   data: {"billableUsd":0.0012,"costCredits":0.004,...}
```

## Blocked projects

When a project crosses its overuse limit, `aiRun` throws `AiBlockedError`. Catch and surface a clean error:

```ts
try {
  await aiRun(c.env, model, { messages });
} catch (e) {
  if (e instanceof AiBlockedError) return ApiErrors.badRequest(c, 'AI usage limit reached');
  throw e;
}
```

## Non-chat Workers AI models

Embeddings, image, STT, TTS through `@cf/*` model ids work too — type the generic to match the model's native shape:

```ts
const emb = await aiRun<{ data: number[][]; shape: number[] }>(
  c.env,
  '@cf/baai/bge-base-en-v1.5',
  { text: ['hello'] },
);
```

Third-party embeddings / image / audio (OpenAI text-embedding, DALL-E, Anthropic vision) are not wired — use CF-hosted equivalents.

Realtime voice + stateful provider APIs (Assistants, Files) are not supported.

## Where `aiRun` runs vs. where it doesn't

`aiRun` works from any top-level worker context — `fetch` handlers, queue consumers, heartbeat handlers, alarms. It does **not** work from inside a Durable Object's `fetch` (DO subrequests bypass the dispatch outbound worker — see `docs/agents.md`). Agents that need LLM calls use the agent runtime's built-in provider config, which routes through the platform fanout instead of `aiRun` directly.

## Don'ts

- Don't bind `AI` in `wrangler.toml`.
- Don't add `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / etc. as project secrets for ordinary LLM work.
- Don't try to shortcut by fetching `poof-ai.internal` with custom headers — the proxy ignores caller-supplied identity.
- Don't import `@cloudflare/voice` (not wired).
