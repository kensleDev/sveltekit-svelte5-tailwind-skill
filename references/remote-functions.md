---
title: "Remote Functions: Type-Safe Server-Client Communication"
version_anchors: ["SvelteKit@2.x", "Svelte@5.x"]
authored: true
origin: self
adapted_from:
  - "sveltejs/kit repository (remote functions documentation)"
last_reviewed: 2025-11-13
summary: "Build type-safe server-client communication with SvelteKit remote functions. Replace form libraries with built-in Form and Command functions, handle data loading with Query, and implement optimistic UI updates with Standard Schema validation."
---

# Remote Functions: Type-Safe Server-Client Communication

Remote functions enable type-safe communication between client and server. They can be **called anywhere in your app**, but always **run on the server**, allowing secure access to server-only modules like databases and environment variables.

## Why Remote Functions?

Remote functions provide better ergonomics than traditional form actions and eliminate the need for form libraries.

**Traditional form actions vs. Remote functions:**

❌ **Traditional approach: Complex callbacks**
```svelte
<script>
  import { enhance } from '$app/forms';
  let submitting = $state(false);

  const handleSubmit = enhance(() => {
    submitting = true;
    return async ({ result, update }) => {
      submitting = false;
      await update();
    };
  });
</script>

<form method="POST" use:handleSubmit>
  <button disabled={submitting}>Submit</button>
</form>
```

✅ **Remote functions: Simple and type-safe**
```svelte
<script>
  import { createContact } from './contact.server';

  const contact = createContact.form();
</script>

<form {...contact.props}>
  <input name="email" />
  <button disabled={contact.submitting}>
    {contact.submitting ? 'Sending...' : 'Submit'}
  </button>
</form>
```

**Key advantages:**
- ✅ Built-in TypeScript support for arguments and return values
- ✅ Automatic validation with Standard Schema (Zod, Valibot)
- ✅ Progressive enhancement (forms work without JavaScript)
- ✅ Single-flight mutations (refresh specific queries instead of all data)
- ✅ Optimistic updates with `.withOverride()`
- ✅ No form library needed

## Setup

Enable remote functions in `svelte.config.js`:

```js
// svelte.config.js
export default {
  kit: {
    experimental: {
      remoteFunctions: true
    }
  }
};
```

## The Four Function Types

### 1. Query - Read Dynamic Data

Query functions read data from the server. Results are cached during page sessions and can be refreshed.

**Basic query:**
```ts
// src/routes/products/products.server.ts
import { query } from '$app/server';
import { db } from '$lib/server/db';

export const getProducts = query(async () => {
  return await db.product.findMany();
});
```

**Use in component:**
```svelte
<script>
  import { getProducts } from './products.server';

  const products = getProducts();
</script>

{#await products}
  <p>Loading...</p>
{:then data}
  <ul>
    {#each data as product}
      <li>{product.name} - ${product.price}</li>
    {/each}
  </ul>
{:catch error}
  <p>Error: {error.message}</p>
{/await}
```

**With Svelte 5 runes:**
```svelte
<script>
  import { getProducts } from './products.server';

  const products = getProducts();
  let items = $state([]);

  $effect(() => {
    products.then(data => items = data);
  });
</script>

<ul>
  {#each items as product}
    <li>{product.name}</li>
  {/each}
</ul>
```

**Refreshing queries:**
```svelte
<script>
  import { getProducts } from './products.server';

  const products = getProducts();

  async function refresh() {
    await products.refresh();
  }
</script>

<button onclick={refresh}>Refresh</button>
```

❌ **Wrong: Calling query on every render**
```svelte
<script>
  import { getProducts } from './products.server';
</script>

{#each getProducts() as product}
  <!-- Creates new request on every render! -->
{/each}
```

✅ **Right: Call once, reuse result**
```svelte
<script>
  import { getProducts } from './products.server';

  const products = getProducts(); // Call once
</script>

{#await products}
  <!-- ... -->
{/await}
```

### 2. Query.batch - Solve N+1 Problem

`Query.batch` groups simultaneous queries into single requests, solving the n+1 problem by collecting multiple calls within the same macrotask.

**Without batching (N+1 problem):**
```svelte
<script>
  import { getProduct } from './products.server';

  let { productIds } = $props();
</script>

<!-- Makes N separate requests! -->
{#each productIds as id}
  {#await getProduct(id) then product}
    <div>{product.name}</div>
  {/await}
{/each}
```

**With batching:**
```ts
// products.server.ts
import { query } from '$app/server';

// Single query batches all IDs together
export const getProduct = query.batch(
  async (ids: string[]) => {
    const products = await db.product.findMany({
      where: { id: { in: ids } }
    });

    // Return Map with results for each ID
    return new Map(products.map(p => [p.id, p]));
  }
);
```

```svelte
<script>
  import { getProduct } from './products.server';

  let { productIds } = $props();
</script>

<!-- All requests batched into single query! -->
{#each productIds as id}
  {#await getProduct(id) then product}
    <div>{product.name}</div>
  {/await}
{/each}
```

**How it works:**
- Multiple calls to `getProduct(id)` within the same macrotask are collected
- SvelteKit calls your function once with all IDs
- Results are distributed to each caller

### 3. Form - Write Data with Forms

Form functions handle form submissions with progressive enhancement. They return an object spreadable onto `<form>` elements.

**Basic form function:**
```ts
// contact.server.ts
import { form } from '$app/server';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email'),
  message: z.string().min(10, 'Message too short')
});

export const createContact = form(schema, async (data) => {
  // data is validated and type-safe
  await db.contact.create({
    data: {
      email: data.email,
      message: data.message
    }
  });

  return { success: true };
});
```

**Use in component:**
```svelte
<script>
  import { createContact } from './contact.server';

  const contact = createContact.form();
</script>

<form {...contact.props}>
  <input
    type="email"
    name="email"
    required
  />
  {#if contact.errors?.email}
    <p class="text-red-500">{contact.errors.email}</p>
  {/if}

  <textarea
    name="message"
    required
  ></textarea>
  {#if contact.errors?.message}
    <p class="text-red-500">{contact.errors.message}</p>
  {/if}

  <button disabled={contact.submitting}>
    {contact.submitting ? 'Sending...' : 'Send'}
  </button>

  {#if contact.result?.success}
    <p class="text-green-500">Message sent!</p>
  {/if}
</form>
```

**Form object properties:**
```ts
contact.props      // Spread onto <form>: action, method, onsubmit
contact.submitting // Boolean: is form submitting?
contact.errors     // Field-level validation errors
contact.result     // Return value from server function
```

**Progressive enhancement:**
- Form works without JavaScript (standard POST request)
- With JavaScript: no page reload, loading states, instant validation feedback
- SvelteKit handles everything automatically

### 4. Command - Imperative Actions

Command functions are like Form but callable from anywhere, not tied to form elements. Useful for actions triggered by buttons, timers, or other events.

**Basic command:**
```ts
// likes.server.ts
import { command } from '$app/server';
import { z } from 'zod';

const schema = z.object({
  postId: z.string()
});

export const addLike = command(schema, async (data, event) => {
  const userId = event.locals.user.id;

  await db.like.create({
    data: {
      postId: data.postId,
      userId
    }
  });

  const count = await db.like.count({
    where: { postId: data.postId }
  });

  return { count };
});
```

**Use in component:**
```svelte
<script>
  import { addLike } from './likes.server';

  let { postId } = $props();
  let likes = $state(0);

  async function handleLike() {
    const result = await addLike({ postId });
    if (result.count) {
      likes = result.count;
    }
  }
</script>

<button onclick={handleLike}>
  ❤️ {likes}
</button>
```

**Difference from Form:**
- Form returns object with `props` to spread on `<form>`
- Command returns callable function directly
- Command can be called from any event handler

### 5. Prerender - Static Build-Time Data

Prerender functions are invoked at build time for static data. Results are cached using the Cache API and cleared on new deployments.

```ts
// config.server.ts
import { prerender } from '$app/server';

export const getConfig = prerender(async () => {
  return {
    siteName: 'My App',
    apiUrl: process.env.API_URL
  };
});
```

```svelte
<script>
  import { getConfig } from './config.server';

  const config = getConfig(); // Runs at build time
</script>

<h1>{config.siteName}</h1>
```

## Validation with Standard Schema

Remote functions support Standard Schema validators like Zod and Valibot.

**Zod validation:**
```ts
import { form } from '$app/server';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Min 8 characters'),
  confirmPassword: z.string()
}).refine(
  data => data.password === data.confirmPassword,
  {
    message: "Passwords don't match",
    path: ['confirmPassword']
  }
);

export const register = form(schema, async (data) => {
  // data.password and data.confirmPassword are validated
  await createUser(data);
  return { success: true };
});
```

**Valibot validation:**
```ts
import { form } from '$app/server';
import * as v from 'valibot';

const schema = v.object({
  email: v.pipe(v.string(), v.email('Invalid email')),
  age: v.pipe(v.number(), v.minValue(18, 'Must be 18+'))
});

export const signup = form(schema, async (data) => {
  // Fully type-safe
  return { success: true };
});
```

**Custom validation errors:**
```ts
// src/hooks.server.ts
export const handleValidationError = ({ error, field }) => {
  // Customize error messages
  return {
    [field]: `Custom error: ${error.message}`
  };
};
```

## Optimistic Updates

Use `.withOverride()` for immediate UI feedback while request processes.

**Optimistic likes:**
```svelte
<script>
  import { addLike } from './likes.server';
  import { getLikes } from './likes.server';

  let { postId } = $props();
  const likesQuery = getLikes(postId);

  let likes = $state(0);

  $effect(() => {
    likesQuery.then(data => likes = data.count);
  });

  async function handleLike() {
    // Show optimistic update immediately
    const optimisticLikes = likes + 1;
    likes = optimisticLikes;

    // Make server request
    const result = await addLike
      .withOverride({ count: optimisticLikes })
      .call({ postId });

    // Update with real count
    likes = result.count;

    // Refresh query cache
    await likesQuery.refresh();
  }
</script>

<button onclick={handleLike}>
  ❤️ {likes}
</button>
```

**Optimistic todo addition:**
```svelte
<script>
  import { addTodo, getTodos } from './todos.server';

  const todosQuery = getTodos();
  let todos = $state([]);
  let optimisticId = $state(null);

  $effect(() => {
    todosQuery.then(data => todos = data);
  });

  async function handleAdd(text) {
    // Show immediately
    const tempId = `temp-${Date.now()}`;
    optimisticId = tempId;
    todos = [...todos, { id: tempId, text, pending: true }];

    try {
      // Add on server
      const result = await addTodo({ text });

      // Replace optimistic with real
      todos = todos.map(t =>
        t.id === tempId ? result.todo : t
      );
    } catch (error) {
      // Revert on failure
      todos = todos.filter(t => t.id !== tempId);
    } finally {
      optimisticId = null;
    }
  }
</script>

{#each todos as todo}
  <div class:opacity-50={todo.pending}>
    {todo.text}
    {#if todo.pending}(saving...){/if}
  </div>
{/each}
```

❌ **Wrong: Not reverting on failure**
```svelte
<script>
  async function handleAdd(text) {
    items = [...items, newItem]; // Added optimistically
    await addItem({ text }); // What if this fails?
    // Item stays in list even on error!
  }
</script>
```

✅ **Right: Revert on failure**
```svelte
<script>
  async function handleAdd(text) {
    const temp = { id: 'temp', text };
    items = [...items, temp];

    try {
      const result = await addItem({ text });
      items = items.map(i => i.id === 'temp' ? result.item : i);
    } catch (error) {
      items = items.filter(i => i.id !== 'temp');
    }
  }
</script>
```

## Coordinating Queries and Mutations

Refresh specific queries after mutations instead of invalidating all data.

**Query + Form coordination:**
```svelte
<script>
  import { getProducts } from './products.server';
  import { createProduct } from './products.server';

  const productsQuery = getProducts();
  const productForm = createProduct.form();

  let products = $state([]);

  $effect(() => {
    productsQuery.then(data => products = data);
  });

  async function handleSuccess() {
    // Refresh only products query
    await productsQuery.refresh();
  }
</script>

<form
  {...productForm.props}
  onsubmit={async (e) => {
    e.preventDefault();
    const result = await productForm.submit();
    if (result.success) {
      await handleSuccess();
    }
  }}
>
  <input name="name" required />
  <button disabled={productForm.submitting}>Add Product</button>
</form>

<ul>
  {#each products as product}
    <li>{product.name}</li>
  {/each}
</ul>
```

**Query + Command coordination:**
```svelte
<script>
  import { getTodos, deleteTodo } from './todos.server';

  const todosQuery = getTodos();
  let todos = $state([]);

  $effect(() => {
    todosQuery.then(data => todos = data);
  });

  async function handleDelete(id) {
    await deleteTodo({ id });
    await todosQuery.refresh(); // Refresh query
  }
</script>

{#each todos as todo}
  <div>
    {todo.text}
    <button onclick={() => handleDelete(todo.id)}>Delete</button>
  </div>
{/each}
```

## Accessing Request Context

Use `getRequestEvent()` to access cookies, headers, and request context.

```ts
// auth.server.ts
import { form, getRequestEvent } from '$app/server';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  password: z.string()
});

export const login = form(schema, async (data) => {
  const event = getRequestEvent();

  const user = await authenticate(data.email, data.password);

  if (!user) {
    throw new Error('Invalid credentials');
  }

  // Set session cookie
  event.cookies.set('session', user.sessionId, {
    path: '/',
    httpOnly: true,
    secure: true,
    sameSite: 'strict'
  });

  return { user };
});
```

**Accessing locals:**
```ts
export const createPost = form(schema, async (data) => {
  const event = getRequestEvent();
  const userId = event.locals.user.id; // From hooks

  const post = await db.post.create({
    data: {
      ...data,
      userId
    }
  });

  return { post };
});
```

## Sensitive Data Handling

Prefix field names with underscore to exclude from client-side form state.

```ts
// payment.server.ts
const schema = z.object({
  amount: z.number(),
  _cardNumber: z.string(), // Won't be sent to client
  _cvv: z.string()         // Won't be sent to client
});

export const processPayment = form(schema, async (data) => {
  // data._cardNumber only exists on server
  await chargeCard(data._cardNumber, data._cvv, data.amount);
  return { success: true };
});
```

## Migration from Form Actions

**Before: Traditional form action**
```ts
// +page.server.ts
import { fail } from '@sveltejs/kit';
import type { Actions } from './$types';

export const actions = {
  default: async ({ request }) => {
    const data = await request.formData();
    const email = data.get('email');

    if (!email) {
      return fail(400, { error: 'Email required' });
    }

    await sendEmail(email);
    return { success: true };
  }
} satisfies Actions;
```

```svelte
<!-- +page.svelte -->
<script>
  import { enhance } from '$app/forms';
  let { form } = $props();
  let submitting = $state(false);

  const handleSubmit = enhance(() => {
    submitting = true;
    return async ({ result, update }) => {
      submitting = false;
      await update();
    };
  });
</script>

<form method="POST" use:handleSubmit>
  <input type="email" name="email" />
  {#if form?.error}
    <p>{form.error}</p>
  {/if}
  <button disabled={submitting}>Submit</button>
</form>
```

**After: Remote function**
```ts
// contact.server.ts
import { form } from '$app/server';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email')
});

export const sendContact = form(schema, async (data) => {
  await sendEmail(data.email);
  return { success: true };
});
```

```svelte
<!-- +page.svelte -->
<script>
  import { sendContact } from './contact.server';

  const contact = sendContact.form();
</script>

<form {...contact.props}>
  <input type="email" name="email" />
  {#if contact.errors?.email}
    <p>{contact.errors.email}</p>
  {/if}
  <button disabled={contact.submitting}>
    {contact.submitting ? 'Sending...' : 'Send'}
  </button>
</form>
```

**What changed:**
- ✅ No more `enhance` callbacks
- ✅ Built-in validation with Zod
- ✅ Type-safe throughout
- ✅ Less boilerplate
- ✅ Better error handling

## TypeScript Support

Remote functions provide full type inference.

**Fully typed query:**
```ts
import { query } from '$app/server';

type Product = {
  id: string;
  name: string;
  price: number;
};

export const getProducts = query(async (): Promise<Product[]> => {
  return await db.product.findMany();
});
```

```svelte
<script lang="ts">
  import { getProducts } from './products.server';

  const products = getProducts();
  // products is Promise<Product[]>
</script>

{#await products then data}
  <!-- data is Product[] -->
  {#each data as product}
    <!-- product is Product -->
    <p>{product.name}: ${product.price}</p>
  {/each}
{/await}
```

**Typed form with Zod:**
```ts
import { form } from '$app/server';
import { z } from 'zod';

const schema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number()
});

export const createUser = form(schema, async (data) => {
  // data is inferred as { name: string; email: string; age: number }
  const user = await db.user.create({ data });
  return { user };
});
```

```svelte
<script lang="ts">
  import { createUser } from './user.server';

  const userForm = createUser.form();
  // userForm.result is { user: User } | undefined
  // userForm.errors is { name?: string; email?: string; age?: string }
</script>
```

## Common Pitfalls

**Pitfall 1: Calling query multiple times**

❌ **Wrong:**
```svelte
<script>
  import { getProduct } from './products.server';
</script>

<!-- Creates new request each time! -->
<h1>{#await getProduct(id) then p}{p.name}{/await}</h1>
<p>{#await getProduct(id) then p}{p.description}{/await}</p>
```

✅ **Right:**
```svelte
<script>
  import { getProduct } from './products.server';

  const product = getProduct(id); // Call once
</script>

{#await product then p}
  <h1>{p.name}</h1>
  <p>{p.description}</p>
{/await}
```

**Pitfall 2: Not handling form errors**

❌ **Wrong:**
```svelte
<form {...contact.props}>
  <input name="email" />
  <button>Submit</button>
  <!-- No error display! -->
</form>
```

✅ **Right:**
```svelte
<form {...contact.props}>
  <input
    name="email"
    class:border-red-500={contact.errors?.email}
  />
  {#if contact.errors?.email}
    <p class="text-red-500">{contact.errors.email}</p>
  {/if}
  <button disabled={contact.submitting}>Submit</button>
</form>
```

**Pitfall 3: Forgetting to enable experimental flag**

If remote functions don't work, check `svelte.config.js`:

```js
export default {
  kit: {
    experimental: {
      remoteFunctions: true // Required!
    }
  }
};
```

**Pitfall 4: Not refreshing queries after mutations**

❌ **Wrong:**
```svelte
<script>
  const items = getItems();

  async function addItem(text) {
    await createItem({ text });
    // Items list won't update!
  }
</script>
```

✅ **Right:**
```svelte
<script>
  const items = getItems();

  async function addItem(text) {
    await createItem({ text });
    await items.refresh(); // Refresh query
  }
</script>
```

## Complete CRUD Example

**Server functions:**
```ts
// todos.server.ts
import { query, form, command } from '$app/server';
import { z } from 'zod';
import { db } from '$lib/server/db';

// Query: Get all todos
export const getTodos = query(async () => {
  return await db.todo.findMany({
    orderBy: { createdAt: 'desc' }
  });
});

// Form: Create todo
const createSchema = z.object({
  text: z.string().min(1, 'Todo cannot be empty')
});

export const createTodo = form(createSchema, async (data) => {
  const todo = await db.todo.create({
    data: { text: data.text, completed: false }
  });
  return { todo };
});

// Command: Toggle completion
const toggleSchema = z.object({
  id: z.string()
});

export const toggleTodo = command(toggleSchema, async (data) => {
  const todo = await db.todo.findUnique({
    where: { id: data.id }
  });

  const updated = await db.todo.update({
    where: { id: data.id },
    data: { completed: !todo.completed }
  });

  return { todo: updated };
});

// Command: Delete todo
export const deleteTodo = command(toggleSchema, async (data) => {
  await db.todo.delete({
    where: { id: data.id }
  });
  return { success: true };
});
```

**Component:**
```svelte
<script>
  import { getTodos, createTodo, toggleTodo, deleteTodo } from './todos.server';

  const todosQuery = getTodos();
  const todoForm = createTodo.form();

  let todos = $state([]);

  $effect(() => {
    todosQuery.then(data => todos = data);
  });

  async function handleSubmit(e) {
    e.preventDefault();
    const result = await todoForm.submit();

    if (result.todo) {
      await todosQuery.refresh();
      todoForm.reset();
    }
  }

  async function handleToggle(id) {
    await toggleTodo({ id });
    await todosQuery.refresh();
  }

  async function handleDelete(id) {
    if (confirm('Delete this todo?')) {
      await deleteTodo({ id });
      await todosQuery.refresh();
    }
  }
</script>

<div class="max-w-2xl mx-auto p-6">
  <h1 class="text-3xl font-bold mb-6">Todos</h1>

  <!-- Create Form -->
  <form
    {...todoForm.props}
    onsubmit={handleSubmit}
    class="mb-6"
  >
    <div class="flex gap-2">
      <input
        type="text"
        name="text"
        placeholder="Add a todo..."
        class="flex-1 rounded border px-4 py-2"
        class:border-red-500={todoForm.errors?.text}
      />
      <button
        type="submit"
        disabled={todoForm.submitting}
        class="rounded bg-blue-500 px-6 py-2 text-white hover:bg-blue-600 disabled:opacity-50"
      >
        {todoForm.submitting ? 'Adding...' : 'Add'}
      </button>
    </div>

    {#if todoForm.errors?.text}
      <p class="mt-1 text-sm text-red-500">{todoForm.errors.text}</p>
    {/if}
  </form>

  <!-- Todo List -->
  <ul class="space-y-2">
    {#each todos as todo}
      <li class="flex items-center gap-3 rounded border p-3">
        <input
          type="checkbox"
          checked={todo.completed}
          onchange={() => handleToggle(todo.id)}
          class="h-5 w-5"
        />
        <span class:line-through={todo.completed} class="flex-1">
          {todo.text}
        </span>
        <button
          onclick={() => handleDelete(todo.id)}
          class="rounded bg-red-500 px-3 py-1 text-sm text-white hover:bg-red-600"
        >
          Delete
        </button>
      </li>
    {/each}

    {#if todos.length === 0}
      <li class="text-center text-gray-500 py-8">
        No todos yet. Add one above!
      </li>
    {/if}
  </ul>
</div>
```

## When to Use Each Type

**Use Query when:**
- Reading data from server
- Need caching across component renders
- Want to refresh data manually
- Solving n+1 problems (use Query.batch)

**Use Form when:**
- Submitting HTML forms
- Need progressive enhancement
- Want built-in validation UI
- Handling file uploads

**Use Command when:**
- Triggering actions from buttons or events
- Don't need form semantics
- Imperative operations (like, delete, toggle)

**Use Prerender when:**
- Data doesn't change between deployments
- Configuration or static content
- Want to reduce server load

## Checklist

- [ ] Enabled `experimental.remoteFunctions` in svelte.config.js
- [ ] Added validation schema (Zod/Valibot)
- [ ] Handled all error states in UI
- [ ] Displayed loading states (submitting, etc.)
- [ ] Refresh queries after mutations
- [ ] Test progressive enhancement (JavaScript disabled)
- [ ] Handle optimistic update failures
- [ ] Use TypeScript for type safety
- [ ] Prefix sensitive fields with underscore

## Next Steps

- Learn form styling in `styling-with-tailwind.md`
- See complete patterns in `integration-patterns.md`
- Optimize queries in `performance-optimization.md`
- Handle errors in `best-practices.md`
