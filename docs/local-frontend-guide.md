# Local Frontend Guide

How to build a frontend that connects to a Poof-hosted backend. This covers SDK setup, wallet authentication, database access, calling PartyServer API routes, and real-time subscriptions.

Use this guide when you're building your own UI (React, Vue, Svelte, React Native, plain HTML) and connecting it to a Poof project created with `generationMode: "backend,policy"` or `"policy"`.

## Contents
- [Prerequisites](#prerequisites)
- [SDK Installation & Setup](#sdk-installation--setup)
- [Wallet Authentication](#wallet-authentication)
- [Two Ways to Access Data](#two-ways-to-access-data)
- [Direct Database Access (get/set/subscribe)](#direct-database-access-getsetsubscribe)
- [Calling PartyServer API Routes](#calling-partyserver-api-routes)
- [Using the Generated Typed SDK](#using-the-generated-typed-sdk)
- [Real-Time Subscriptions (React)](#real-time-subscriptions-react)
- [Special Utilities & Gotchas](#special-utilities--gotchas)
- [Complete React Example](#complete-react-example)
- [Complete Vanilla JS Example](#complete-vanilla-js-example)

## Prerequisites

1. A Poof project with a backend — created via `create_project` with `generationMode: "backend,policy"` or `"policy"`. See [backend-only.md](backend-only.md).
2. Connection info from `get_project_status` — you need `tarobaseAppId`, `backendUrl`, and the API URLs.

## SDK Installation & Setup

### Install

```bash
npm install @pooflabs/web buffer
```

`buffer` is a required polyfill for `@pooflabs/web` and Solana libraries in the browser.

### Initialize

The SDK must be initialized before any auth or data operations. Do this once at app startup.

```typescript
// MUST be first — before any @pooflabs/web imports execute
import { Buffer } from 'buffer';
globalThis.Buffer = Buffer;
window.Buffer = Buffer;

import { init } from '@pooflabs/web';

// connectionInfo comes from get_project_status
// Pick the environment you're connecting to: draft, preview, or production
const env = connectionInfo.draft; // or connectionInfo.preview, connectionInfo.production
await init({
  appId: env.tarobaseAppId,                  // required — per-environment app ID
  authApiUrl: connectionInfo.authApiUrl,     // 'https://auth.tarobase.com'
  apiUrl: connectionInfo.apiUrl,             // 'https://api.tarobase.com'
  wsApiUrl: connectionInfo.wsUrl,            // 'wss://api.tarobase.com/ws/v2'
  authMethod: 'phantom',                    // wallet auth provider
  skipBackendInit: true,                     // frontend-only — no server keypair
});
```

> **Which environment?** Use `draft` during development (free, simulated blockchain). Use `preview` for mainnet testing with real tokens (limited to allowlisted wallets). Use `production` for live. Each has its own separate data.

### Vite Config

If using Vite, add the Buffer polyfill alias:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  resolve: {
    alias: { buffer: 'buffer/' },
  },
  define: {
    global: 'globalThis',
  },
});
```

### Environment Variables (optional)

Instead of hardcoding connection info, use env vars:

```env
# Pick draft, preview, or production from connectionInfo
VITE_TAROBASE_APP_ID=<connectionInfo.draft.tarobaseAppId>
VITE_PARTYSERVER_URL=<connectionInfo.draft.backendUrl>
# Shared URLs from connectionInfo
VITE_API_URL=<connectionInfo.apiUrl>
VITE_WS_API_URL=<connectionInfo.wsUrl>
VITE_AUTH_API_URL=<connectionInfo.authApiUrl>
```

## Wallet Authentication

Poof uses Solana wallet authentication. The `@pooflabs/web` SDK handles the full flow — wallet connection, message signing, and JWT token management.

### React (useAuth hook)

```typescript
import { useAuth, getIdToken } from '@pooflabs/web';

function App() {
  const { user, loading, login, logout } = useAuth();

  if (loading) return <div>Loading...</div>;

  if (!user) {
    return <button onClick={login}>Connect Wallet</button>;
  }

  return (
    <div>
      <p>Connected: {user.address}</p>
      <button onClick={logout}>Disconnect</button>
    </div>
  );
}
```

### Vanilla JS (onAuthStateChanged)

```typescript
import { login, logout, onAuthStateChanged, getCurrentUser } from '@pooflabs/web';

onAuthStateChanged((user) => {
  if (user) {
    console.log('Logged in:', user.address);
  } else {
    console.log('Logged out');
  }
});

// Trigger login (opens Phantom wallet prompt)
document.getElementById('login-btn').addEventListener('click', () => login());
```

### Getting a JWT Token (for API calls)

```typescript
import { getIdToken } from '@pooflabs/web';

const token = await getIdToken();
// Use in Authorization header: `Bearer ${token}`
```

The token is a Cognito JWT (~1 hour expiry) containing the user's wallet address and app ID. Refresh it before each batch of API calls.

### Auth is Required for Protected Writes

If your policy has access rules that check `@user.address` (anything other than `"true"` for write rules), the user **must** be logged in before writing data. Without auth, the policy engine has no identity to check and will reject the write.

Public-read collections (`"read": "true"`) can be read without auth.

## Two Ways to Access Data

Your frontend can access data through two paths:

| Path | When to use | How |
|------|------------|-----|
| **Direct database** | CRUD on collections defined in the policy | `get()`, `set()`, `subscribe()` from `@pooflabs/web` |
| **PartyServer API routes** | Custom logic, aggregations, third-party APIs, admin operations | `fetch()` to `backendUrl/api/...` with auth headers |

Most apps use both — direct database for simple reads/writes, and API routes for anything that needs server-side logic.

## Direct Database Access (get/set/subscribe)

The `@pooflabs/web` SDK provides `get`, `set`, and `subscribe` for direct database operations against collections defined in your policy.

### Write

```typescript
import { set } from '@pooflabs/web';

// Create or update a document
await set('notes/note-123', {
  title: 'My Note',
  content: 'Hello world',
  authorAddress: user.address,
  createdAt: Date.now(),
});
```

The write goes through the policy engine — access rules are checked against the authenticated user's wallet address.

### Read

```typescript
import { get } from '@pooflabs/web';

// Read a single document
const note = await get('notes/note-123');
console.log(note.title); // 'My Note'

// Read a collection (returns object keyed by ID)
const allNotes = await get('notes');
```

### Delete

```typescript
import { set } from '@pooflabs/web';

// Delete by setting to null
await set('notes/note-123', null);
```

### Subscribe (real-time)

```typescript
import { subscribe } from '@pooflabs/web';

// Subscribe to changes on a document
const unsubscribe = subscribe('notes/note-123', (data) => {
  console.log('Note updated:', data);
});

// Subscribe to an entire collection
const unsubscribe = subscribe('notes', (data) => {
  console.log('All notes:', data);
});

// Cleanup when done
unsubscribe();
```

## Calling PartyServer API Routes

The PartyServer hosts custom API routes at `backendUrl` from `connectionInfo`. These routes can have server-side logic, access secrets, call external APIs, and perform admin operations.

### API Client Pattern

```typescript
const BACKEND_URL = connectionInfo.backendUrl; // e.g. 'https://{appId}-api.poof.new'

// Public endpoint (no auth)
const res = await fetch(`${BACKEND_URL}/api/notes`);
const { success, data } = await res.json();

// Authenticated endpoint
import { getIdToken, getCurrentUser } from '@pooflabs/web';

const token = await getIdToken();
const user = getCurrentUser();

const res = await fetch(`${BACKEND_URL}/api/notes`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
    'X-Wallet-Address': user.address,
  },
  body: JSON.stringify({ title: 'New Note', content: 'Hello' }),
});
const { success, data } = await res.json();
```

### Response Format

All PartyServer routes return:

```typescript
{
  success: boolean;
  data?: any;          // on success
  error?: {            // on failure
    code: string;
    message: string;
  };
  timestamp: number;
  requestId?: string;
}
```

### Reusable API Client

For convenience, create a small helper (adapted from the Poof v3 template):

```typescript
// lib/api-client.ts
const BACKEND_URL = 'https://<your-appId>-api.poof.new';

class ApiClient {
  constructor(private token?: string, private walletAddress?: string) {}

  private async request<T>(method: string, path: string, body?: any): Promise<T> {
    const headers: Record<string, string> = { 'Content-Type': 'application/json' };
    if (this.token) headers['Authorization'] = `Bearer ${this.token}`;
    if (this.walletAddress) headers['X-Wallet-Address'] = this.walletAddress;

    const res = await fetch(`${BACKEND_URL}${path}`, {
      method,
      headers,
      ...(body ? { body: JSON.stringify(body) } : {}),
    });
    const json = await res.json();
    if (!json.success) throw new Error(json.error?.message || `API error: ${res.status}`);
    return json.data;
  }

  get<T = any>(path: string) { return this.request<T>('GET', path); }
  post<T = any>(path: string, body?: any) { return this.request<T>('POST', path, body); }
  put<T = any>(path: string, body?: any) { return this.request<T>('PUT', path, body); }
  delete<T = any>(path: string) { return this.request<T>('DELETE', path); }
}

// Public client (no auth)
export const api = new ApiClient();

// Create authenticated client after login
export function createAuthenticatedApiClient(token: string, walletAddress: string) {
  return new ApiClient(token, walletAddress);
}
```

## Using the Generated Typed SDK

When Poof builds your project, it generates a typed SDK from the policy — collection-specific functions with TypeScript types. This is the recommended way to interact with data for type safety.

### Getting the SDK

Download via `download_code` → `get_download_url` (requires a credit purchase), then extract:

```
src/lib/db-client.ts              # Shared utilities (Time, Address, Increment, Token)
src/lib/collections/notes.ts      # Per-collection: setNotes, getNotes, subscribeNotes, types
src/lib/collections/users.ts
```

Copy these into your local project. They import from `@pooflabs/web` internally.

### Using Collection Functions

```typescript
import { setNotes, getNotes, deleteNotes, subscribeManyNotes } from './lib/collections/notes';
import { Time, Address } from './lib/db-client';

// Write (typed — TypeScript catches missing/wrong fields)
await setNotes('note-123', {
  title: 'My Note',
  content: 'Hello',
  author: Address.publicKey(user.address),  // wrap addresses
  createdAt: Time.Now,                      // server timestamp
});

// Read
const note = await getNotes('note-123');
console.log(note?.title);  // typed as NotesResponse

// Delete
await deleteNotes('note-123');
```

### Special Utilities from db-client.ts

| Utility | Usage | Why |
|---------|-------|-----|
| `Time.Now` | `{ createdAt: Time.Now }` | Server-side timestamp (Unix seconds). **Never use `Date.now()`** for timestamp fields. |
| `Address.publicKey(key)` | `{ owner: Address.publicKey(addr) }` | Wraps Solana addresses for policy enforcement. Required for `Address` type fields. |
| `Increment.by(n)` | `{ likes: Increment.by(1) }` | Atomic increment/decrement — no read-modify-write race conditions. |
| `Token.amount(name, amt)` | `Token.amount('SOL', 1.5)` | Handles decimal conversion (SOL=9 decimals, USDC=6). |
| `setMany([...])` | `await setMany([buildNotes(...), buildUsers(...)])` | Atomic batch writes. |

See [database-sdk.md](database-sdk.md) for the full SDK reference.

## Real-Time Subscriptions (React)

For React apps, use this `useRealtimeData` hook (from the Poof v3 template):

```typescript
// hooks/use-realtime-data.ts
import { useState, useEffect } from 'react';

export function useRealtimeData<T>(
  subscribeFn: (callback: (data: T) => void, ...args: any[]) => Promise<() => void>,
  enabled: boolean,
  ...args: any[]
) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    if (!enabled) {
      setData(null);
      setLoading(false);
      return;
    }

    setLoading(true);
    let unsubscribe: (() => void) | undefined;

    subscribeFn((newData: T) => {
      setData(newData);
      setLoading(false);
    }, ...args)
      .then((unsub) => { unsubscribe = unsub; })
      .catch((err) => {
        setError(err instanceof Error ? err : new Error(String(err)));
        setLoading(false);
      });

    return () => { unsubscribe?.(); };
  }, [subscribeFn, enabled, ...args]);

  return { data, loading, error };
}
```

### Usage with Generated SDK

```typescript
import { useRealtimeData } from './hooks/use-realtime-data';
import { subscribeNotes, subscribeManyNotes, NotesResponse } from './lib/collections/notes';

// Single document
const { data: note, loading } = useRealtimeData<NotesResponse | null>(
  subscribeNotes,
  !!noteId,      // enabled flag — subscription only activates when true
  noteId
);

// Filtered collection
const { data: notes } = useRealtimeData<NotesResponse[]>(
  subscribeManyNotes,
  true,
  "where author = 'alice' order by createdAt desc limit 20"
);
```

### Usage with Raw subscribe

```typescript
import { subscribe } from '@pooflabs/web';

// Wrap raw subscribe in the hook-compatible signature
function subscribeToNotes(callback: (data: any) => void) {
  return Promise.resolve(subscribe('notes', callback));
}

const { data: allNotes, loading } = useRealtimeData(subscribeToNotes, true);
```

## Special Utilities & Gotchas

### Timestamps are Unix Seconds

Poof timestamps are Unix **seconds**, not milliseconds. When displaying:

```typescript
const date = new Date(note.createdAt * 1000); // multiply by 1000
```

When writing, always use `Time.Now` (from the generated SDK) or let the backend set it. Never use `Date.now()` — it returns milliseconds and isn't server-authoritative.

### Token Decimals

SOL has 9 decimals, USDC has 6. When writing token amounts:

```typescript
import { Token } from './lib/db-client';
await setStakes('stake-1', { amount: Token.amount('SOL', 1.5) }); // handles decimals
```

When reading, divide by the decimal factor:

```typescript
const humanAmount = rawAmount / Math.pow(10, 9); // SOL
const humanAmount = rawAmount / Math.pow(10, 6); // USDC
```

### On-Chain vs Off-Chain Collections

- **Off-chain** (most collections) — fast, free, supports all access rules
- **On-chain** — requires `"read": "true"` (blockchain data is public), costs ~0.002-0.01 SOL per record, limited to 4-6 fields

Off-chain collections can read on-chain data in hooks, but not vice versa.

### No Arrays or Objects in Fields

Poof fields are flat: `String`, `Address`, `UInt`, `Int`, `Bool`. For lists, use sub-collections:

```
// Instead of: posts/$postId { tags: string[] }
// Use:        posts/$postId/tags/$tagId { name: String }
```

### Auth Loading State

Always check `loading` before rendering auth-dependent UI to avoid flicker:

```typescript
const { user, loading } = useAuth();
if (loading) return <Spinner />;
if (!user) return <LoginButton />;
return <Dashboard />;
```

## Complete React Example

A minimal React + Vite app connected to a Poof backend:

```typescript
// main.tsx
import { Buffer } from 'buffer';
globalThis.Buffer = Buffer;
window.Buffer = Buffer;

import { init } from '@pooflabs/web';
import { createRoot } from 'react-dom/client';
import App from './App';

await init({
  appId: import.meta.env.VITE_TAROBASE_APP_ID,
  authApiUrl: 'https://auth.tarobase.com',
  apiUrl: 'https://api.tarobase.com',
  wsApiUrl: 'wss://api.tarobase.com/ws/v2',
  authMethod: 'phantom',
  skipBackendInit: true,
});

createRoot(document.getElementById('root')!).render(<App />);
```

```typescript
// App.tsx
import { useAuth, getIdToken } from '@pooflabs/web';
import { get, set } from '@pooflabs/web';
import { useState, useEffect } from 'react';

const BACKEND_URL = import.meta.env.VITE_PARTYSERVER_URL;

export default function App() {
  const { user, loading, login, logout } = useAuth();
  const [notes, setNotes] = useState<any[]>([]);

  // Load notes from direct database access
  useEffect(() => {
    if (!user) return;
    get('notes').then((data) => {
      if (data && typeof data === 'object') {
        setNotes(Object.entries(data).map(([id, v]) => ({ id, ...v })));
      }
    });
  }, [user]);

  // Create note via PartyServer API route
  async function createNote(title: string, content: string) {
    const token = await getIdToken();
    const res = await fetch(`https://${BACKEND_URL}/api/notes`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
        'X-Wallet-Address': user!.address,
      },
      body: JSON.stringify({ title, content }),
    });
    const { data } = await res.json();
    return data;
  }

  // Or create note via direct database write
  async function createNoteDirect(title: string, content: string) {
    const noteId = `note-${Date.now()}`;
    await set(`notes/${noteId}`, {
      title,
      content,
      authorAddress: user!.address,
      createdAt: Date.now(),
    });
  }

  if (loading) return <p>Loading...</p>;
  if (!user) return <button onClick={login}>Connect Wallet</button>;

  return (
    <div>
      <p>Connected: {user.address}</p>
      <button onClick={logout}>Disconnect</button>
      <h2>Notes ({notes.length})</h2>
      {notes.map((n) => <div key={n.id}>{n.title}</div>)}
    </div>
  );
}
```

## Complete Vanilla JS Example

No framework — plain HTML + JS with `@pooflabs/web`:

```html
<!DOCTYPE html>
<html>
<head><title>Notes App</title></head>
<body>
  <button id="login-btn">Connect Wallet</button>
  <button id="logout-btn" style="display:none">Disconnect</button>
  <p id="status">Not connected</p>
  <div id="notes"></div>

  <script type="module">
    import { Buffer } from 'buffer';
    globalThis.Buffer = Buffer;
    window.Buffer = Buffer;

    import { init, login, logout, onAuthStateChanged, get, set, getIdToken } from '@pooflabs/web';

    await init({
      appId: '<tarobaseAppId>',
      authApiUrl: 'https://auth.tarobase.com',
      apiUrl: 'https://api.tarobase.com',
      wsApiUrl: 'wss://api.tarobase.com/ws/v2',
      authMethod: 'phantom',
      skipBackendInit: true,
    });

    let currentUser = null;

    onAuthStateChanged((user) => {
      currentUser = user;
      document.getElementById('login-btn').style.display = user ? 'none' : '';
      document.getElementById('logout-btn').style.display = user ? '' : 'none';
      document.getElementById('status').textContent = user
        ? `Connected: ${user.address}`
        : 'Not connected';
      if (user) loadNotes();
    });

    document.getElementById('login-btn').addEventListener('click', () => login());
    document.getElementById('logout-btn').addEventListener('click', () => logout());

    async function loadNotes() {
      const data = await get('notes');
      const el = document.getElementById('notes');
      if (!data) { el.innerHTML = '<p>No notes</p>'; return; }
      el.innerHTML = Object.entries(data)
        .map(([id, n]) => `<div><strong>${n.title}</strong><p>${n.content}</p></div>`)
        .join('');
    }
  </script>
</body>
</html>
```
