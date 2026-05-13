# Queues

Poof Cloud Queues are the only supported async primitive for durable backend jobs. Use them for: retries, batching of similar work, decoupled producers/consumers, dead-letter-queue handling, or anything that exceeds a single Worker request's CPU/wall-clock budget.

Do **not** create polling loops, public "run queue" endpoints, `setInterval` daemons, or third-party schedulers.

## File layout

```text
partyserver/
├── queues.json                # queue declarations (consumed by Poof at deploy)
├── src/queues/index.ts        # platform registry — registers /__internal/queues/:queueName
└── src/queues/<queue-name>.ts # one consumer per queue
```

Don't edit `src/queues/index.ts` past the registry call. Per-queue handlers live in their own files.

## Declaring a queue

```jsonc
// partyserver/queues.json
{
  "queues": [
    {
      "name": "send-emails",
      "maxBatchSize": 10,
      "maxBatchTimeoutSec": 5,
      "maxRetries": 3,
      "deadLetterQueue": "send-emails-dlq",
      "consumer": "src/queues/send-emails.ts"
    }
  ]
}
```

Poof reads `queues.json` at deploy time. It provisions the Cloudflare Queue, emits the producer + consumer bindings into `wrangler.toml`, and creates the DLQ if declared. Tenant code never touches the wrangler binding directly.

## Producing (enqueueing)

From any top-level handler (`fetch`, heartbeat, alarm, another queue consumer):

```ts
import { enqueueQueue } from '../lib/poof-queue.js';

app.post('/api/welcome', async (c) => {
  const { walletAddress } = await validatePoofAuth(c);
  const { email } = await c.req.json();
  const job = await enqueueQueue(c.env, 'send-emails', { email, walletAddress });
  return sendSuccess(c, { jobId: job.id });
});
```

`enqueueQueue` returns `{ id, status }`. Caller can ignore the id (fire-and-forget) or surface it to the UI for status polling.

## Consuming

```ts
// partyserver/src/queues/send-emails.ts
import type { MessageBatch } from '@cloudflare/workers-types';

type SendEmailJob = { email: string; walletAddress: string };

export async function handleSendEmailsQueue(batch: MessageBatch<SendEmailJob>, env: any) {
  for (const msg of batch.messages) {
    try {
      await fetch('https://api.sendgrid.com/...', { /* ... */ });
      msg.ack();
    } catch (err) {
      // Retry up to maxRetries; goes to DLQ after.
      msg.retry();
    }
  }
}
```

Then wire it once in `src/queues/index.ts` (only this file imports the per-queue handler):

```ts
import { handleSendEmailsQueue } from './send-emails.js';

const handlers = {
  'send-emails': handleSendEmailsQueue,
} satisfies Record<string, QueueHandler>;
```

## Job-status tracking (optional)

Queue producers return `{ id, status }`. Status is tracked via the `QueueJobTracker` DO (auto-bound when queues are configured). To expose job status to the frontend, add a tenant `/api/*` route that reads tracker state — gate it with `validatePoofAuth` so other users can't poll your jobs.

## What runs in a queue handler

Queue consumers are **top-level worker invocations** — they get `env`, can call `aiRun`, can invoke agents via `runAgent`, can write to Tarobase, can call MCP via the platform fanout. The only thing they can't do is hold long-lived state across messages (use Tarobase / DO storage for that).

## Don'ts

- Don't expose `/__internal/queues/:queueName` to public traffic — it's reserved for the Cloudflare queue dispatcher.
- Don't enqueue from a Durable Object's `fetch()`. DOs *can* call `env.QUEUE.send` (binding methods bypass outbound), but the cleaner pattern is "DO returns instructions → tenant route enqueues."
- Don't catch+ack inside the consumer when the upstream call failed. `msg.retry()` is what lets Cloudflare's retry/DLQ machinery work.
- Don't run multiple consumers off the same queue. One queue → one consumer file.

## Promotion from local to deployed

`queues.json` is part of the uploaded artifact. Poof reads it at deploy time, provisions the Cloudflare Queue (idempotent), and registers the consumer. The UI's **Queues** tab reads `partyserver/queues.json` from the deployed artifact — what you declare locally is what shows up.
