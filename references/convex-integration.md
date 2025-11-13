---
title: "Convex Backend Integration with SvelteKit and Svelte 5"
version_anchors: ["SvelteKit@2.x", "Svelte@5.x", "Convex@1.x"]
authored: true
origin: self
adapted_from:
  - "get-convex/convex-js repository (Convex client documentation)"
last_reviewed: 2025-11-13
summary: "Integrate Convex backend with SvelteKit and Svelte 5 runes. Learn queries, mutations, actions, real-time subscriptions, authentication, and when to use Convex vs SvelteKit remote functions."
---

# Convex Backend Integration with SvelteKit and Svelte 5

Convex provides a complete backend (database + server functions + real-time) that integrates seamlessly with SvelteKit and Svelte 5 runes.

## Why Convex with SvelteKit?

Convex replaces traditional backends with a type-safe, real-time alternative that works perfectly with Svelte 5's reactive system.

**Traditional Backend vs. Convex:**

❌ **Traditional: Multiple services**
```
- Database (PostgreSQL/MongoDB)
- API routes (+page.server.ts)
- WebSocket server for real-time
- Authentication service
- File storage (S3)
- Separate deployment
```

✅ **Convex: All-in-one**
```
- Database (built-in)
- Server functions (queries, mutations, actions)
- Real-time subscriptions (automatic)
- Authentication (built-in)
- File storage (built-in)
- Single deployment
```

**Key advantages:**
- ✅ **Type-safe end-to-end** - TypeScript from database to UI
- ✅ **Real-time by default** - All queries auto-subscribe
- ✅ **Optimistic updates** - Built-in patterns
- ✅ **No ORM needed** - Direct database access with validation
- ✅ **Works with Svelte 5 runes** - Reactive queries integrate perfectly
- ✅ **Edge deployment** - Fast worldwide

## Setup

### Installation

```bash
# Install Convex packages
bun add convex convex-svelte

# Install concurrently for dev script
bun add -D concurrently
```

### Configuration

**1. Create `convex.json` in project root:**

```json
{
  "$schema": "https://raw.githubusercontent.com/get-convex/convex-backend/refs/heads/main/npm-packages/convex/schemas/convex.schema.json",
  "functions": "src/convex/"
}
```

**2. Update `package.json` dev scripts:**

```json
{
  "scripts": {
    "dev": "concurrently --names \"SVELTE,CONVEX\" -c yellow,magenta \"bun run dev:sv\" \"bun run dev:cx\"",
    "dev:sv": "vite dev",
    "dev:cx": "convex dev"
  }
}
```

**3. Update `svelte.config.js` with Convex alias:**

```js
// svelte.config.js
export default {
  kit: {
    alias: {
      $convex: './src/convex'
    }
  }
};
```

**4. Setup Convex in root layout:**

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import { PUBLIC_CONVEX_URL } from '$env/static/public';
  import { setupConvex } from 'convex-svelte';
  import '../app.css';

  const { children } = $props();

  setupConvex(PUBLIC_CONVEX_URL);
</script>

{@render children()}
```

**5. (Optional) Add Convex cursor rules:**

```bash
curl "https://www.davis7.sh/sv/rules?rule=convex" -o .cursor/rules/convex.mdc
```

**6. Initialize Convex:**

```bash
# Run dev to initialize Convex project
bun run dev:cx
```

This creates:
- `src/convex/` - Your backend functions
- `.env.local` - Environment variables with Convex URL

**Environment variables:**

```bash
# .env.local (auto-generated)
PUBLIC_CONVEX_URL=https://your-deployment.convex.cloud
```

## Convex vs Remote Functions

**When to use what:**

| Feature | Convex | Remote Functions | Traditional Forms |
|---------|--------|------------------|-------------------|
| Real-time | ✅ Built-in | ❌ No | ❌ No |
| Type safety | ✅ End-to-end | ✅ Yes | ⚠️ Manual |
| Optimistic updates | ✅ Built-in | ✅ Manual | ⚠️ Manual |
| Database | ✅ Included | ❌ Bring your own | ❌ Bring your own |
| Deployment | Single | SvelteKit | SvelteKit |
| Progressive enhancement | ⚠️ Requires JS | ✅ Works without JS | ✅ Works without JS |
| Learning curve | Medium | Low | Low |

**Decision matrix:**

**Use Convex when:**
- Building real-time features (chat, collaboration, live updates)
- Want type-safe database access
- Need automatic data synchronization
- Building complex data relationships
- Want single deployment for frontend + backend

**Use Remote Functions when:**
- Simple forms without real-time needs
- Need progressive enhancement (works without JavaScript)
- Already have existing backend
- Simple CRUD operations

**Use both when:**
- Convex for core app data (real-time, complex)
- Remote functions for simple contact forms, newsletter signups

## Schema Definition

Define your database schema in Convex:

```ts
// src/convex/schema.ts
import { defineSchema, defineTable } from 'convex/server';
import { v } from 'convex/values';

export default defineSchema({
  todos: defineTable({
    text: v.string(),
    completed: v.boolean(),
    userId: v.id('users'),
    createdAt: v.number()
  })
    .index('by_user', ['userId'])
    .index('by_created', ['createdAt']),

  users: defineTable({
    name: v.string(),
    email: v.string(),
    tokenIdentifier: v.string()
  })
    .index('by_token', ['tokenIdentifier'])
    .index('by_email', ['email'])
});
```

**Schema validation:**
- Type-safe at compile time
- Runtime validation
- Indexes for performance
- Relationships via IDs
- Located in `src/convex/` (not root `convex/`)

## Queries with Svelte 5 Runes

Convex queries integrate perfectly with Svelte 5's reactive system using `convex-svelte`.

### Basic Query Pattern

```ts
// src/convex/todos.ts
import { query } from './_generated/server';
import { v } from 'convex/values';

export const list = query({
  args: {},
  handler: async (ctx) => {
    return await ctx.db.query('todos').collect();
  }
});

export const get = query({
  args: { id: v.id('todos') },
  handler: async (ctx, args) => {
    return await ctx.db.get(args.id);
  }
});
```

```svelte
<!-- +page.svelte -->
<script>
  import { useQuery } from 'convex-svelte';
  import { api } from '$convex/_generated/api';

  // Real-time reactive query
  const todos = useQuery(api.todos.list, {});
</script>

{#if $todos === undefined}
  <p>Loading...</p>
{:else}
  <ul>
    {#each $todos as todo}
      <li>{todo.text}</li>
    {/each}
  </ul>
{/if}
```

**Benefits of `convex-svelte`:**
- ✅ Automatic subscriptions and cleanup
- ✅ Svelte store integration (use `$` prefix)
- ✅ Built-in loading and error states
- ✅ No manual `$effect` needed

### Query with Arguments

```ts
// src/convex/todos.ts
export const getByUser = query({
  args: { userId: v.id('users') },
  handler: async (ctx, args) => {
    return await ctx.db
      .query('todos')
      .withIndex('by_user', (q) => q.eq('userId', args.userId))
      .collect();
  }
});
```

```svelte
<script>
  import { useQuery } from 'convex-svelte';
  import { api } from '$convex/_generated/api';

  let userId = $state('user-123');

  // Automatically re-subscribes when userId changes
  const userTodos = useQuery(api.todos.getByUser, { userId });
</script>

{#if $userTodos === undefined}
  <p>Loading...</p>
{:else}
  <ul>
    {#each $userTodos as todo}
      <li>{todo.text}</li>
    {/each}
  </ul>
{/if}
```

## Mutations with Svelte 5

Mutations modify data and trigger real-time updates using `convex-svelte`.

### Basic Mutation

```ts
// src/convex/todos.ts
import { mutation } from './_generated/server';
import { v } from 'convex/values';

export const create = mutation({
  args: {
    text: v.string()
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Not authenticated');

    const todoId = await ctx.db.insert('todos', {
      text: args.text,
      completed: false,
      userId: identity.subject,
      createdAt: Date.now()
    });

    return todoId;
  }
});

export const toggle = mutation({
  args: { id: v.id('todos') },
  handler: async (ctx, args) => {
    const todo = await ctx.db.get(args.id);
    if (!todo) throw new Error('Todo not found');

    await ctx.db.patch(args.id, {
      completed: !todo.completed
    });
  }
});

export const remove = mutation({
  args: { id: v.id('todos') },
  handler: async (ctx, args) => {
    await ctx.db.delete(args.id);
  }
});
```

### Using Mutations in Components

```svelte
<script>
  import { useQuery, useMutation } from 'convex-svelte';
  import { api } from '$convex/_generated/api';

  const todos = useQuery(api.todos.list, {});
  const createTodo = useMutation(api.todos.create);
  const toggleTodo = useMutation(api.todos.toggle);
  const removeTodo = useMutation(api.todos.remove);

  let newTodoText = $state('');

  async function handleCreate() {
    if (!newTodoText.trim()) return;

    await createTodo({ text: newTodoText });
    newTodoText = '';
  }

  async function handleToggle(id) {
    await toggleTodo({ id });
  }

  async function handleDelete(id) {
    if (confirm('Delete this todo?')) {
      await removeTodo({ id });
    }
  }
</script>

<div class="max-w-2xl mx-auto p-6">
  <h1 class="text-3xl font-bold mb-6">Todos</h1>

  <!-- Create Form -->
  <form onsubmit={(e) => { e.preventDefault(); handleCreate(); }} class="mb-6">
    <div class="flex gap-2">
      <input
        type="text"
        bind:value={newTodoText}
        placeholder="Add a todo..."
        class="flex-1 px-4 py-2 border rounded focus:ring-2 focus:ring-blue-500"
      />
      <button
        type="submit"
        disabled={submitting}
        class="px-6 py-2 bg-blue-500 text-white rounded hover:bg-blue-600
          disabled:opacity-50"
      >
        {submitting ? 'Adding...' : 'Add'}
      </button>
    </div>
  </form>

  <!-- Todo List -->
  {#if todosQuery.loading}
    <p class="text-gray-500">Loading todos...</p>
  {:else if todosQuery.error}
    <p class="text-red-500">Error: {todosQuery.error.message}</p>
  {:else if todosQuery.data}
    <ul class="space-y-2">
      {#each todosQuery.data as todo}
        <li
          class="flex items-center gap-3 p-3 border rounded
            bg-white hover:bg-gray-50 transition-colors"
        >
          <input
            type="checkbox"
            checked={todo.completed}
            onchange={() => handleToggle(todo._id)}
            class="h-5 w-5 rounded border-gray-300
              text-blue-600 focus:ring-blue-500"
          />
          <span class="flex-1 {todo.completed ? 'line-through text-gray-500' : ''}">
            {todo.text}
          </span>
          <button
            onclick={() => handleDelete(todo._id)}
            class="px-3 py-1 text-sm bg-red-500 text-white rounded
              hover:bg-red-600 transition-colors"
          >
            Delete
          </button>
        </li>
      {/each}

      {#if todosQuery.data.length === 0}
        <li class="text-center text-gray-500 py-8">
          No todos yet. Add one above!
        </li>
      {/if}
    </ul>
  {/if}
</div>
```

## Optimistic Updates

Convex supports optimistic updates for instant UI feedback.

```ts
// src/lib/convex-mutation.svelte.ts
import { convex } from '$lib/convex';
import type { FunctionReference } from 'convex/server';

export function useConvexMutation<T>(mutation: FunctionReference<'mutation'>) {
  let loading = $state(false);
  let error = $state<Error | undefined>(undefined);

  async function execute(args: any, optimisticUpdate?: () => void) {
    loading = true;
    error = undefined;

    // Apply optimistic update immediately
    if (optimisticUpdate) {
      optimisticUpdate();
    }

    try {
      const result = await convex.mutation(mutation, args);
      return result;
    } catch (err) {
      error = err as Error;
      throw err;
    } finally {
      loading = false;
    }
  }

  return { execute, loading, error };
}
```

**Using optimistic updates:**

```svelte
<script>
  import { useConvexMutation } from '$lib/convex-mutation.svelte';
  import { api } from '../../convex/_generated/api';

  let todos = $state([]);
  let optimisticId = $state(null);

  const createMutation = useConvexMutation(api.todos.create);

  async function handleCreate(text) {
    const tempId = `temp-${Date.now()}`;
    const tempTodo = {
      _id: tempId,
      text,
      completed: false,
      pending: true
    };

    try {
      await createMutation.execute(
        { text },
        () => {
          // Optimistic update
          todos = [...todos, tempTodo];
          optimisticId = tempId;
        }
      );
    } catch (error) {
      // Revert on failure
      todos = todos.filter(t => t._id !== tempId);
    } finally {
      optimisticId = null;
    }
  }
</script>
```

## Actions for Server-Side Operations

Actions run on the Convex backend and can call third-party APIs.

```ts
// convex/notifications.ts
import { action } from './_generated/server';
import { v } from 'convex/values';

export const sendEmail = action({
  args: {
    to: v.string(),
    subject: v.string(),
    body: v.string()
  },
  handler: async (ctx, args) => {
    // Call external email service
    const response = await fetch('https://api.sendgrid.com/v3/mail/send', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.SENDGRID_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        personalizations: [{
          to: [{ email: args.to }]
        }],
        from: { email: 'noreply@example.com' },
        subject: args.subject,
        content: [{
          type: 'text/plain',
          value: args.body
        }]
      })
    });

    if (!response.ok) {
      throw new Error('Failed to send email');
    }

    // Can also trigger mutations
    await ctx.runMutation(api.notifications.create, {
      email: args.to,
      subject: args.subject,
      sentAt: Date.now()
    });

    return { success: true };
  }
});
```

**Using actions:**

```svelte
<script>
  import { convex } from '$lib/convex';
  import { api } from '../../convex/_generated/api';

  let sending = $state(false);

  async function sendNotification() {
    sending = true;
    try {
      await convex.action(api.notifications.sendEmail, {
        to: 'user@example.com',
        subject: 'Welcome!',
        body: 'Thanks for signing up.'
      });
      alert('Email sent!');
    } catch (error) {
      alert('Failed to send email');
    } finally {
      sending = false;
    }
  }
</script>
```

## Authentication

Convex provides built-in authentication.

### Setup Clerk (Recommended)

```bash
npm install @clerk/clerk-js
```

**Configure Clerk provider:**

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { ClerkProvider } from '@clerk/clerk-js';
  import { PUBLIC_CLERK_PUBLISHABLE_KEY } from '$env/static/public';
  import { convex } from '$lib/convex';
  import { browser } from '$app/environment';

  if (browser) {
    const clerk = new ClerkProvider(PUBLIC_CLERK_PUBLISHABLE_KEY);

    clerk.addListener(async ({ session }) => {
      const token = await session?.getToken({ template: 'convex' });
      convex.setAuth(token || null);
    });
  }
</script>

<slot />
```

### Protected Queries and Mutations

```ts
// convex/todos.ts
import { query, mutation } from './_generated/server';

export const list = query({
  args: {},
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Not authenticated');

    return await ctx.db
      .query('todos')
      .withIndex('by_user', (q) => q.eq('userId', identity.subject))
      .collect();
  }
});

export const create = mutation({
  args: { text: v.string() },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Not authenticated');

    return await ctx.db.insert('todos', {
      text: args.text,
      completed: false,
      userId: identity.subject,
      createdAt: Date.now()
    });
  }
});
```

### Auth UI Components

```svelte
<!-- src/lib/components/AuthGate.svelte -->
<script>
  import { SignInButton, UserButton, useUser } from '@clerk/clerk-js';

  const user = useUser();
</script>

{#if $user}
  <div class="flex items-center gap-4">
    <span>Welcome, {$user.firstName}!</span>
    <UserButton />
  </div>
  <slot />
{:else}
  <div class="max-w-md mx-auto mt-16 p-6 text-center">
    <h1 class="text-2xl font-bold mb-4">Sign in to continue</h1>
    <SignInButton mode="modal">
      <button class="px-6 py-2 bg-blue-500 text-white rounded hover:bg-blue-600">
        Sign In
      </button>
    </SignInButton>
  </div>
{/if}
```

**Use in pages:**

```svelte
<!-- +page.svelte -->
<script>
  import AuthGate from '$lib/components/AuthGate.svelte';
  import { useConvexQuery } from '$lib/convex-query.svelte';
  import { api } from '../../convex/_generated/api';

  const todosQuery = useConvexQuery(api.todos.list, {});
</script>

<AuthGate>
  <!-- This content only visible when authenticated -->
  {#if todosQuery.data}
    <ul>
      {#each todosQuery.data as todo}
        <li>{todo.text}</li>
      {/each}
    </ul>
  {/if}
</AuthGate>
```

## Real-Time Subscriptions

All Convex queries are real-time by default.

### Presence System

```ts
// convex/presence.ts
import { mutation, query } from './_generated/server';
import { v } from 'convex/values';

export const updatePresence = mutation({
  args: {
    roomId: v.string(),
    data: v.any()
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Not authenticated');

    const existing = await ctx.db
      .query('presence')
      .withIndex('by_room_user', (q) =>
        q.eq('roomId', args.roomId).eq('userId', identity.subject)
      )
      .first();

    if (existing) {
      await ctx.db.patch(existing._id, {
        data: args.data,
        lastSeen: Date.now()
      });
    } else {
      await ctx.db.insert('presence', {
        roomId: args.roomId,
        userId: identity.subject,
        data: args.data,
        lastSeen: Date.now()
      });
    }
  }
});

export const getRoomPresence = query({
  args: { roomId: v.string() },
  handler: async (ctx, args) => {
    const fiveMinutesAgo = Date.now() - 5 * 60 * 1000;

    return await ctx.db
      .query('presence')
      .withIndex('by_room', (q) => q.eq('roomId', args.roomId))
      .filter((q) => q.gt(q.field('lastSeen'), fiveMinutesAgo))
      .collect();
  }
});
```

**Using presence:**

```svelte
<script>
  import { convex } from '$lib/convex';
  import { useConvexQuery } from '$lib/convex-query.svelte';
  import { api } from '../../convex/_generated/api';

  let roomId = $state('room-1');
  const presenceQuery = useConvexQuery(api.presence.getRoomPresence, { roomId });

  // Update presence every 30 seconds
  $effect(() => {
    const interval = setInterval(() => {
      convex.mutation(api.presence.updatePresence, {
        roomId,
        data: { status: 'active' }
      });
    }, 30000);

    return () => clearInterval(interval);
  });
</script>

<div class="flex gap-2">
  <span class="font-semibold">Online:</span>
  {#if presenceQuery.data}
    <span>{presenceQuery.data.length} users</span>
  {/if}
</div>
```

## File Storage

Convex provides built-in file storage.

### Upload Files

```ts
// convex/files.ts
import { mutation } from './_generated/server';
import { v } from 'convex/values';

export const generateUploadUrl = mutation({
  args: {},
  handler: async (ctx) => {
    return await ctx.storage.generateUploadUrl();
  }
});

export const saveFile = mutation({
  args: {
    storageId: v.id('_storage'),
    name: v.string(),
    type: v.string()
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Not authenticated');

    const fileId = await ctx.db.insert('files', {
      storageId: args.storageId,
      name: args.name,
      type: args.type,
      userId: identity.subject,
      uploadedAt: Date.now()
    });

    return fileId;
  }
});
```

**File upload component:**

```svelte
<script>
  import { convex } from '$lib/convex';
  import { api } from '../../convex/_generated/api';

  let uploading = $state(false);
  let uploadProgress = $state(0);

  async function handleFileUpload(e) {
    const file = e.target.files?.[0];
    if (!file) return;

    uploading = true;
    uploadProgress = 0;

    try {
      // 1. Generate upload URL
      const uploadUrl = await convex.mutation(api.files.generateUploadUrl, {});

      // 2. Upload file
      const result = await fetch(uploadUrl, {
        method: 'POST',
        headers: { 'Content-Type': file.type },
        body: file
      });

      const { storageId } = await result.json();
      uploadProgress = 100;

      // 3. Save file metadata
      await convex.mutation(api.files.saveFile, {
        storageId,
        name: file.name,
        type: file.type
      });

      alert('File uploaded successfully!');
    } catch (error) {
      console.error('Upload failed:', error);
      alert('Upload failed');
    } finally {
      uploading = false;
    }
  }
</script>

<div class="space-y-4">
  <input
    type="file"
    onchange={handleFileUpload}
    disabled={uploading}
    class="block w-full text-sm file:mr-4 file:rounded file:border-0
      file:bg-blue-500 file:px-4 file:py-2 file:text-white
      hover:file:bg-blue-600"
  />

  {#if uploading}
    <div class="space-y-2">
      <div class="h-2 w-full overflow-hidden rounded-full bg-gray-200">
        <div
          class="h-full bg-blue-500 transition-all"
          style="width: {uploadProgress}%"
        ></div>
      </div>
      <p class="text-sm text-gray-600">Uploading... {uploadProgress}%</p>
    </div>
  {/if}
</div>
```

### Display Files

```ts
// convex/files.ts
export const getFileUrl = query({
  args: { storageId: v.id('_storage') },
  handler: async (ctx, args) => {
    return await ctx.storage.getUrl(args.storageId);
  }
});
```

```svelte
<script>
  import { useConvexQuery } from '$lib/convex-query.svelte';
  import { api } from '../../convex/_generated/api';

  let { storageId } = $props();

  const urlQuery = useConvexQuery(api.files.getFileUrl, { storageId });
</script>

{#if urlQuery.data}
  <img src={urlQuery.data} alt="Uploaded file" class="max-w-md rounded" />
{/if}
```

## SSR Considerations

Convex queries run client-side by default. For SSR, use SvelteKit's load functions.

### Server-Side Query

```ts
// +page.server.ts
import { ConvexHttpClient } from 'convex/browser';
import { PUBLIC_CONVEX_URL } from '$env/static/public';
import { api } from '../../convex/_generated/api';
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async () => {
  const convexClient = new ConvexHttpClient(PUBLIC_CONVEX_URL);

  const todos = await convexClient.query(api.todos.list, {});

  return { todos };
};
```

```svelte
<!-- +page.svelte -->
<script>
  import { useConvexQuery } from '$lib/convex-query.svelte';
  import { api } from '../../convex/_generated/api';

  let { data } = $props();

  // Hydrate with SSR data, then subscribe to updates
  let todos = $state(data.todos);

  $effect(() => {
    const unsubscribe = convex.onUpdate(api.todos.list, {}, (newTodos) => {
      todos = newTodos;
    });

    return unsubscribe;
  });
</script>

<ul>
  {#each todos as todo}
    <li>{todo.text}</li>
  {/each}
</ul>
```

❌ **Wrong: Client-only data**
```svelte
<script>
  // Page shows loading state on SSR
  const todosQuery = useConvexQuery(api.todos.list, {});
</script>

{#if todosQuery.loading}
  <p>Loading...</p> <!-- Shows this on initial SSR! -->
{/if}
```

✅ **Right: SSR + hydration**
```svelte
<script>
  let { data } = $props();
  let todos = $state(data.todos); // SSR data available immediately

  $effect(() => {
    // Subscribe to real-time updates after hydration
    const unsubscribe = convex.onUpdate(api.todos.list, {}, (newTodos) => {
      todos = newTodos;
    });
    return unsubscribe;
  });
</script>
```

## Common Patterns

### Pagination

```ts
// convex/todos.ts
export const paginatedList = query({
  args: {
    paginationOpts: paginationOptsValidator
  },
  handler: async (ctx, args) => {
    return await ctx.db
      .query('todos')
      .order('desc')
      .paginate(args.paginationOpts);
  }
});
```

```svelte
<script>
  import { convex } from '$lib/convex';
  import { api } from '../../convex/_generated/api';

  let page = $state({ continueCursor: null });
  let todos = $state([]);
  let hasMore = $state(true);

  async function loadMore() {
    const result = await convex.query(api.todos.paginatedList, {
      paginationOpts: { cursor: page.continueCursor, numItems: 20 }
    });

    todos = [...todos, ...result.page];
    hasMore = result.isDone === false;
    page.continueCursor = result.continueCursor;
  }

  $effect(() => {
    loadMore();
  });
</script>

<ul>
  {#each todos as todo}
    <li>{todo.text}</li>
  {/each}
</ul>

{#if hasMore}
  <button onclick={loadMore} class="mt-4 px-4 py-2 bg-blue-500 text-white rounded">
    Load More
  </button>
{/if}
```

### Search

```ts
// convex/todos.ts
export const search = query({
  args: { searchTerm: v.string() },
  handler: async (ctx, args) => {
    return await ctx.db
      .query('todos')
      .withSearchIndex('search_text', (q) =>
        q.search('text', args.searchTerm)
      )
      .collect();
  }
});
```

```svelte
<script>
  import { useConvexQuery } from '$lib/convex-query.svelte';
  import { api } from '../../convex/_generated/api';

  let searchTerm = $state('');

  // Automatically re-queries when searchTerm changes
  let searchResults = $derived(
    searchTerm.length > 0
      ? useConvexQuery(api.todos.search, { searchTerm })
      : { data: [] }
  );
</script>

<input
  type="search"
  bind:value={searchTerm}
  placeholder="Search todos..."
  class="w-full px-4 py-2 border rounded"
/>

{#if searchResults.data}
  <ul>
    {#each searchResults.data as todo}
      <li>{todo.text}</li>
    {/each}
  </ul>
{/if}
```

## Common Pitfalls

**Pitfall 1: Creating client in components**

❌ **Wrong:**
```svelte
<script>
  import { ConvexClient } from 'convex/browser';
  const convex = new ConvexClient(PUBLIC_CONVEX_URL); // New client every render!
</script>
```

✅ **Right:**
```ts
// src/lib/convex.ts - Singleton
export const convex = new ConvexClient(PUBLIC_CONVEX_URL);
```

**Pitfall 2: Not cleaning up subscriptions**

❌ **Wrong:**
```svelte
<script>
  $effect(() => {
    convex.onUpdate(api.todos.list, {}, (todos) => {
      // Subscription never cleaned up!
    });
  });
</script>
```

✅ **Right:**
```svelte
<script>
  $effect(() => {
    const unsubscribe = convex.onUpdate(api.todos.list, {}, (todos) => {
      // ...
    });

    return unsubscribe; // Cleanup on component unmount
  });
</script>
```

**Pitfall 3: Forgetting authentication checks**

❌ **Wrong:**
```ts
export const list = query({
  handler: async (ctx) => {
    // Returns ALL todos for ALL users!
    return await ctx.db.query('todos').collect();
  }
});
```

✅ **Right:**
```ts
export const list = query({
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Not authenticated');

    return await ctx.db
      .query('todos')
      .withIndex('by_user', (q) => q.eq('userId', identity.subject))
      .collect();
  }
});
```

**Pitfall 4: Using queries in mutations**

❌ **Wrong:**
```ts
export const create = mutation({
  handler: async (ctx, args) => {
    // Can't call queries from mutations!
    const todos = await ctx.db.query('todos').collect();
  }
});
```

✅ **Right:**
```ts
export const create = mutation({
  handler: async (ctx, args) => {
    // Use direct database access
    const todo = await ctx.db.get(todoId);
  }
});
```

## Checklist

- [ ] Installed `convex` and `convex-svelte` packages
- [ ] Installed `concurrently` for dev scripts
- [ ] Created `convex.json` in project root with `src/convex/` path
- [ ] Updated `package.json` with `dev`, `dev:sv`, `dev:cx` scripts
- [ ] Added `$convex` alias to `svelte.config.js`
- [ ] Called `setupConvex(PUBLIC_CONVEX_URL)` in root `+layout.svelte`
- [ ] Ran `bun run dev:cx` to initialize Convex project
- [ ] Defined schema in `src/convex/schema.ts` with proper indexes
- [ ] Set up authentication (Clerk or other provider)
- [ ] Protected queries and mutations check `ctx.auth.getUserIdentity()`
- [ ] Using `useQuery()` and `useMutation()` from `convex-svelte`
- [ ] Accessing queries with `$` prefix (e.g., `$todos`)
- [ ] Using SSR load functions with `ConvexHttpClient` for initial data
- [ ] File uploads use `generateUploadUrl` pattern
- [ ] Optimistic updates handle failures

## Next Steps

**To complete Convex setup, run:**
```bash
bun run dev:cx
```

**Additional resources:**
- Explore Convex dashboard at https://dashboard.convex.dev
- Read full Convex docs at https://docs.convex.dev
- Read convex-svelte docs at https://github.com/xstevenyung/convex-svelte
- Learn about indexes and query performance
- Implement scheduled functions for cron jobs
- Add webhooks for external integrations
- See `remote-functions.md` for SvelteKit-only patterns
- See `forms-and-actions.md` for progressive enhancement
