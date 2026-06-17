# Poof-Native AI: `aiRun`

`aiRun()` is the helper for one-shot AI calls from a top-level handler (fetch route, queue consumer, heartbeat task, alarm). It routes through Poof's metered AI proxy and charges usage against project credits (`$15 = 50 credits`, exact CF cost × 1.05). Chat-compatible models go through Cloudflare AI Gateway; Cloudflare-hosted embeddings, image input/vision/OCR, classifiers, image generation, STT, and TTS go through Workers AI's native endpoint. No tenant-held provider keys are needed; Poof attaches sealed script identity at the edge.

For multi-turn LLM loops with tool calling (web search, browser, structured output), use Agents — they have their own LLM provider config that handles sessions, tools, and streaming. See `docs/agents.md` and the decision table in `local-backend-guide.md#picking-the-right-primitive`.

For ordinary chat completions, do **not** import `openai` / `@anthropic-ai/sdk` / `@ai-sdk/*`, bind `AI` directly in `wrangler.toml`, or call provider URLs.

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

Chat-compatible providers return OpenAI Chat Completions shape — `.choices[0].message.content` for text, `.choices[0].message.tool_calls` for tool calls.

Native Cloudflare Workers AI models do **not** all return chat shape. Embeddings, vision/image input, classifiers, image generation, STT, and TTS use their model-native shapes. Type the generic to the model you call and read the fields documented below.

## Picking a model

| Need | Model |
|---|---|
| General-purpose default | `anthropic/claude-sonnet-4-6` |
| Cheap / fast | `anthropic/claude-haiku-4-5` |
| Hardest reasoning | `anthropic/claude-opus-4-8` |
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

## Tool calls and options

OpenAI-format tool calling works through chat-compatible providers. Each loop iteration is another billed `aiRun` call, so persist the tool result into `messages` and make the follow-up call explicitly:

```ts
const tools = [{
  type: 'function',
  function: {
    name: 'get_weather',
    description: 'Get weather for a city',
    parameters: {
      type: 'object',
      properties: { city: { type: 'string' } },
      required: ['city'],
    },
  },
}];

const result = await aiRun(c.env, 'openai/gpt-4o-mini', { messages, tools });
const message = result.choices[0].message;
```

Pass model inputs such as `messages`, `tools`, `image`, `prompt`, `audio`, or `text` in the third argument. The fourth `options` argument is filtered before it reaches the gateway. Supported model options are:

- `stream`, `stream_options`
- `max_tokens`, `max_completion_tokens`
- `temperature`, `top_p`, `n`, `stop`, `seed`
- `frequency_penalty`, `presence_penalty`, `logit_bias`, `logprobs`, `top_logprobs`
- `response_format`, `tools`, `tool_choice`, `parallel_tool_calls`
- `user`, `reasoning_effort`

Poof-only option: `includeUsage`. It does not reach the model; it asks Poof to return usage metadata or append the final streaming usage event. Unknown option keys are silently dropped, so do not hide required model inputs in `options`.

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

## Native Workers AI models

Native Cloudflare Workers AI model ids (`@cf/*`, or `workers-ai/@cf/*`) route through Workers AI. Text-only chat models can use the chat-completions compatibility path, while embeddings, image input/vision, OCR-style text extraction from images, classifiers, object detection, image generation, STT, and TTS use Workers AI's native endpoint. The proxy unwraps Cloudflare's envelope and returns the model's native output shape directly.

```ts
const emb = await aiRun<{ data: number[][]; shape: number[] }>(
  c.env,
  '@cf/baai/bge-base-en-v1.5',
  { text: ['hello'] },
);
```

Common native shapes:

- Embeddings: `{ data, shape }`
- Gemma vision chat / OCR: OpenAI-style `choices`, but prefer `extractWorkersAiText(result)` because some Workers AI vision models expose `response`, `description`, or `text`.
- LLaVA image-to-text: `{ description }`
- Image classifiers / object detectors: model-native label or detection arrays.
- Whisper STT: `{ text }`
- Image/audio generation and TTS: `ArrayBuffer`

### Image input / vision

Use Gemma vision chat when you want multimodal chat, image descriptions, or OCR-style visible text extraction:

```ts
import {
  aiRun,
  extractWorkersAiText,
  imageDataUrl,
  imageUrlPart,
  textPart,
  type WorkersAiVisionOutput,
} from '../lib/poof-ai.js';

const result = await aiRun<WorkersAiVisionOutput>(
  c.env,
  '@cf/google/gemma-4-26b-a4b-it',
  {
    messages: [{
      role: 'user',
      content: [
        textPart('Describe this image and extract any visible totals.'),
        imageUrlPart(imageDataUrl(base64Png, 'image/png')),
      ],
    }],
  },
  { max_completion_tokens: 128 },
);

const description = extractWorkersAiText(result);
```

For OCR-style text extraction, use the same model and a stricter prompt:

```ts
const result = await aiRun<WorkersAiVisionOutput>(
  c.env,
  '@cf/google/gemma-4-26b-a4b-it',
  {
    messages: [{
      role: 'user',
      content: [
        textPart('Read all visible text exactly. Preserve line breaks. If no text is visible, return an empty string.'),
        imageUrlPart(imageDataUrl(base64Png, 'image/png')),
      ],
    }],
  },
  { max_completion_tokens: 256 },
);

const visibleText = extractWorkersAiText(result);
```

For byte-oriented image models such as LLaVA, classifiers, or object detectors, pass a number array. Number arrays expand quickly inside JSON, so guard image size before calling:

```ts
import {
  aiRun,
  imageBytesToNumberArray,
  POOF_AI_MAX_NUMBER_ARRAY_IMAGE_BYTES,
} from '../lib/poof-ai.js';

if (uploadedImage.size > POOF_AI_MAX_NUMBER_ARRAY_IMAGE_BYTES) {
  return ApiErrors.badRequest(c, 'Image is too large for Workers AI');
}

const bytes = await uploadedImage.arrayBuffer();
const caption = await aiRun<{ description?: string }>(
  c.env,
  '@cf/llava-hf/llava-1.5-7b-hf',
  {
    image: imageBytesToNumberArray(bytes),
    prompt: 'Generate a concise caption for this image.',
    max_tokens: 128,
  },
);
```

### Image generation and audio

```ts
const imageBytes = await aiRun<ArrayBuffer>(
  c.env,
  '@cf/black-forest-labs/flux-1-schnell',
  { prompt: 'a sunset over Lisbon rooftops', steps: 4 },
);

const transcript = await aiRun<{ text: string }>(
  c.env,
  '@cf/openai/whisper-large-v3-turbo',
  { audio: [...new Uint8Array(await audioFile.arrayBuffer())] },
);
```

Keep image payloads small enough that the full JSON request stays under Poof's 8 MiB proxy limit. Data URLs/base64 fit vision-chat inputs; decimal `number[]` payloads are much larger, so the helper's default number-array cap is intentionally lower.

Third-party provider-native embeddings, image generation/editing, audio, and video endpoints (for example OpenAI text embeddings, DALL-E, provider-native Anthropic vision/file APIs, Stability, etc.) are not wired through `aiRun` — use Cloudflare-hosted native Workers AI models for those use cases.

Realtime voice + stateful provider APIs (Assistants, Files) are not supported.

## Where `aiRun` runs vs. where it doesn't

`aiRun` works from any top-level worker context — `fetch` handlers, queue consumers, heartbeat handlers, alarms. It does **not** work from inside a Durable Object's `fetch` (DO subrequests bypass the dispatch outbound worker — see `docs/agents.md`). Agents that need LLM calls use the agent runtime's built-in provider config, which routes through the platform fanout instead of `aiRun` directly.

## Don'ts

- Don't bind `AI` in `wrangler.toml`.
- Don't add `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / etc. as project secrets for ordinary LLM work.
- Don't try to shortcut by fetching `poof-ai.internal` with custom headers — the proxy ignores caller-supplied identity.
- Don't import `@cloudflare/voice` (not wired).
- Don't parse every result as `choices[0].message.content`; native Workers AI models can return `{ description }`, `{ text }`, `{ data, shape }`, label arrays, or binary `ArrayBuffer`.
