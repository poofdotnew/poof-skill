# Database SDK — db-client & Collections

When the Poof AI builds a project, it auto-generates a typed database SDK from your policy. This SDK is the **only** way to interact with Poof Cloud data — never use internal `@pooflabs/*` functions directly.

## Contents
- [File Structure](#file-structure)
- [What Gets Generated Per Collection](#what-gets-generated-per-collection)
- [Shared Utilities (db-client.ts)](#shared-utilities-db-clientts)
- [Frontend Usage (React)](#frontend-usage-react)
- [Backend Usage (PartyServer/Hono)](#backend-usage-partyserverhono)
- [Generated Files Are Read-Only](#generated-files-are-read-only)
- [For External Agents — Using the Database SDK](#for-external-agents--using-the-database-sdk)

## File Structure

The SDK is generated in two locations — one for frontend, one for backend:

```
# Frontend (client-side, React)
src/lib/db-client.ts              # Shared utilities, setMany, special types
src/lib/collections/posts.ts      # Per-collection: set, get, subscribe, types
src/lib/collections/users.ts
src/lib/collections/stakes.ts

# Backend (server-side, PartyServer/Hono)
partyserver/src/db-client.ts
partyserver/src/collections/posts.ts
partyserver/src/collections/users.ts
partyserver/src/collections/stakes.ts
```

**Frontend** collections use `@pooflabs/web` under the hood (subscriptions, wallet-signed writes).
**Backend** collections use `@pooflabs/server` (server-signed writes with `PROJECT_VAULT_PRIVATE_KEY`).

## What Gets Generated Per Collection

For a policy collection like `posts/$postId`, the SDK generates:

```typescript
// src/lib/collections/posts.ts (frontend) or partyserver/src/collections/posts.ts (backend)

// --- Types ---
export interface PostsRequest {
  title: string;
  content: string;
  author: AddressType;       // Address → AddressType for writes, use Address.publicKey()
  createdAt?: number | TimeOperation | IncrementOperation | TokenAmount;  // use Time.Now
  likes?: number | TimeOperation | IncrementOperation | TokenAmount;
}

export interface PostsResponse {
  id: string;
  title: string;
  content: string;
  author: string;            // Address → plain string (base58) in responses
  createdAt: number;
  likes: number;
  tarobase_created_at: number;  // auto-added metadata
}

// --- Write ---
export async function setPosts(postId: string, data: PostsRequest): Promise<boolean>;
export async function deletePosts(postId: string): Promise<boolean>;
export function buildPosts(postId: string, data: PostsRequest): DocumentOperation;  // for setMany()

// --- Read ---
export async function getPosts(postId: string): Promise<PostsResponse | null>;
export async function getManyPosts(filter?: string): Promise<PostsResponse[]>;

// --- Subscribe (real-time) ---
export function subscribePosts(callback: (data: PostsResponse | null) => void, postId: string): Promise<() => Promise<void>>;
export function subscribeManyPosts(callback: (data: PostsResponse[]) => void, filter?: string): Promise<() => Promise<void>>;
```

### Naming Convention

Function names use `${operation}${HierarchicalName}`:
- Collection `posts/$postId` → `setPosts`, `getPosts`, `subscribePosts`, `subscribeManyPosts`
- Types: `PostsRequest`, `PostsResponse`

### Nested Collections

For `users/$userId/stakes/$tokenMint`, names are hierarchical and functions take parent IDs:

```typescript
// Hierarchical name: UsersStakes
export async function setUsersStakes(userId: string, tokenMint: string, data: UsersStakesRequest): Promise<boolean>;
export async function getUsersStakes(userId: string, tokenMint: string): Promise<UsersStakesResponse | null>;
export function subscribeUsersStakes(callback: (data: UsersStakesResponse | null) => void, userId: string, tokenMint: string): Promise<() => Promise<void>>;
export function subscribeManyUsersStakes(callback: (data: UsersStakesResponse[]) => void, userId: string, filter?: string): Promise<() => Promise<void>>;
```

## Shared Utilities (db-client.ts)

```typescript
import { setMany, Time, Increment, Token, Address } from '@/lib/db-client';
```

| Utility | Usage | Description |
|---------|-------|-------------|
| `Time.Now` | `{ createdAt: Time.Now }` | Server-side timestamp placeholder. **Never use `Date.now()`** for timestamp fields. |
| `Increment.by(n)` | `{ likes: Increment.by(1) }` | Atomic increment/decrement. |
| `Token.amount(name, amt)` | `Token.amount('SOL', 1.5)` | Handles decimal conversion (SOL=9, USDC=6). |
| `Address.publicKey(key)` | `{ owner: Address.publicKey(pubkey) }` | Solana address for writes. |
| `setMany([...])` | `await setMany([buildPost(...), buildComment(...)])` | Atomic batch operations. All items must be same type (all onchain OR all offchain). See [set-many.md](agent-use/set-many.md) for composition patterns (transfer+guard, swap+price check, etc.) and failure semantics. |

## Frontend Usage (React)

```typescript
import { useRealtimeData } from '@/hooks/use-realtime-data';
import { subscribePosts, subscribeManyPosts, PostsResponse } from '@/lib/collections/posts';
import { setPosts, deletePosts } from '@/lib/collections/posts';
import { Time, Address } from '@/lib/db-client';

// Subscribe to a single item (real-time)
const { data: post, loading } = useRealtimeData<PostsResponse | null>(
  subscribePosts,
  !!postId,     // enabled flag (required)
  postId        // ...args passed to subscribe callback
);

// Subscribe to a filtered collection
const { data: posts } = useRealtimeData<PostsResponse[]>(
  subscribeManyPosts,
  true,
  "where author = 'alice' order by createdAt desc limit 20"
);

// Write
await setPosts('post-123', {
  title: 'Hello',
  content: 'World',
  author: Address.publicKey(walletAddress),
  createdAt: Time.Now,
});
```

## Backend Usage (PartyServer/Hono)

```typescript
import { getPosts, setPosts, deletePosts } from './collections/posts';
import { Time, Address } from './db-client';

// Read (direct, not subscription)
const post = await getPosts('post-123');

// Write (signed by PROJECT_VAULT_PRIVATE_KEY)
await setPosts('post-123', {
  title: 'Hello',
  content: 'World',
  author: Address.publicKey(vaultAddress),
  createdAt: Time.Now,
});
```

**Backend writes use the vault key** — the policy must allow `@constants.PROJECT_VAULT_ADDRESS` for the operations the backend performs.

### Multi-Wallet Clients

For server apps that need to operate as multiple wallets (e.g., a vault wallet + an admin wallet):

```typescript
import { createWalletClient } from '@pooflabs/server';

const vault = await createWalletClient({ keypair: process.env.VAULT_KEY! });
const admin = await createWalletClient({ keypair: process.env.ADMIN_KEY! });

await vault.set('markets/123', { outcome: 'YES' });
await admin.set('adminLogs/456', { action: 'resolve' });
console.log(vault.address); // vault wallet's public key
```

Each client has the same operations as the default SDK (`set`, `get`, `signMessage`, etc.) but signs as its own wallet. Create once and reuse across requests.

## Generated Files Are Read-Only

The db-client and collections files are **generated from the policy** by the Poof AI. They should be treated as read-only:

- **Do not modify** generated collection files — your changes will be overwritten on the next policy update.
- **Do regenerate** after any policy change by sending a chat message like: *"Regenerate the db-client and collection files to match the updated policy."*
- The Poof AI automatically regenerates these files when it modifies the policy during a `poof iterate` / `poof chat send` interaction.

## For External Agents — Using the Database SDK

If you're building an agent that needs to interact with a Poof project's database:

### Option 1: Let Poof Handle Everything (Recommended)

Use `poof build --mode full` or `poof build --mode ui,policy` — the Poof AI generates the UI, policy, AND the typed SDK. Your agent just sends `poof iterate` / `poof chat send` messages describing what to build.

### Option 2: Policy-Only + Download SDK

For agents that build their own frontend/backend but want Poof's database:

1. **Create a project** with `poof build --mode policy`
2. **Chat** to define your data model via `poof iterate` / `poof chat send` — the AI generates the policy
3. **Download the code** via `poof deploy download` or `poof deploy download-url`
4. **Extract** `src/lib/db-client.ts` + `src/lib/collections/` (for frontend) or `partyserver/src/db-client.ts` + `partyserver/src/collections/` (for backend)
5. **Copy** those files into your own project
6. **After policy changes**, re-download and re-extract — the generated files will reflect the updated schema

> **Auth requirement:** If your policy has access rules that check caller identity (anything other than `true` for write rules), your frontend must integrate Poof wallet auth via `@pooflabs/web` so writes are wallet-signed. Without authenticated identity, the policy engine will reject writes. This only applies to frontend collections — backend collections use server-signed writes with `PROJECT_VAULT_PRIVATE_KEY` and don't need user login.
>
> See [local-frontend-guide.md](local-frontend-guide.md) for complete SDK setup, auth integration, and usage patterns in a local frontend.

```bash
# 1. Create policy-only project
poof build --mode policy -m 'Create collections for: users/$userId (name String, bio String?, avatar String?, createdAt UInt), posts/$postId (title String, content String, author Address, createdAt UInt, likes UInt?)'

# 2. Wait for build to complete (poof build waits automatically)

# 3. Download and extract SDK files
poof deploy download -p <project-id>
# Or get a download URL:
poof deploy download-url -p <project-id> --task <taskId>
# Extract: src/lib/db-client.ts + src/lib/collections/*
```

### Option 3: Full Generation + Selective Use

Create a `full` project, let Poof build everything, then download and cherry-pick just the database SDK files you need. This gives you the most complete SDK since the AI has full context of how the collections interact.
