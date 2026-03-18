# TanStack Start — Server Functions Reference

## Overview

Server functions (`createServerFn`) are the primary mechanism for running server-side code in TanStack Start. They act as type-safe RPC calls — the client calls them, and they execute exclusively on the server.

## API

### createServerFn(options?)

```typescript
import { createServerFn } from '@tanstack/react-start'

const myFn = createServerFn({ method: 'GET' })  // or 'POST'
  .inputValidator(schema)     // optional: validate input
  .middleware([middleware])    // optional: add middleware
  .handler(async (ctx) => {   // required: the function logic
    // ctx.data — validated input data
    // ctx.context — middleware context
    return result
  })
```

### Options

- `method`: `'GET'` (default) or `'POST'`
  - `GET` — for reads, data fetching, idempotent operations
  - `POST` — for mutations, writes, side effects

### Chaining API

The API uses a builder pattern:

```typescript
createServerFn({ method: 'POST' })
  .inputValidator(schema)     // Step 1: Validate input (optional)
  .middleware([mw1, mw2])     // Step 2: Add middleware (optional)
  .handler(async (ctx) => {}) // Step 3: Define handler (required)
```

## Input Validation

### With Zod

```typescript
import { z } from 'zod'

export const createUser = createServerFn({ method: 'POST' })
  .inputValidator(z.object({
    name: z.string().min(1),
    email: z.string().email(),
    age: z.number().min(0),
  }))
  .handler(async ({ data }) => {
    // data is typed: { name: string; email: string; age: number }
    return db.users.create({ data })
  })
```

### With Custom Validator

```typescript
export const updatePost = createServerFn({ method: 'POST' })
  .inputValidator((data: { id: string; title: string }) => data)
  .handler(async ({ data }) => {
    return db.posts.update({ where: { id: data.id }, data: { title: data.title } })
  })
```

## Calling Server Functions

### From Route Loaders

```tsx
const fetchPosts = createServerFn().handler(async () => {
  return db.posts.findMany()
})

export const Route = createFileRoute('/posts')({
  loader: () => fetchPosts(),
  component: PostList,
})

function PostList() {
  const posts = Route.useLoaderData()
  return posts.map(p => <div key={p.id}>{p.title}</div>)
}
```

### From Components (with useServerFn)

```tsx
import { useServerFn } from '@tanstack/react-start'

function MyComponent() {
  const createPost = useServerFn(createPostFn)

  const handleSubmit = async (formData: FormData) => {
    const result = await createPost({
      data: {
        title: formData.get('title') as string,
        body: formData.get('body') as string,
      },
    })
  }
}
```

### From Components (with React Query)

```tsx
function PostList() {
  const getPosts = useServerFn(getPostsFn)

  const { data, isLoading } = useQuery({
    queryKey: ['posts'],
    queryFn: () => getPosts(),
  })

  if (isLoading) return <div>Loading...</div>
  return data.map(p => <div key={p.id}>{p.title}</div>)
}
```

### Direct Call (Server-Side)

```typescript
// From another server function or server-only code
const user = await getCurrentUser()
```

## Redirects

Throw a `redirect` to navigate the user:

```typescript
import { redirect } from '@tanstack/react-router'

export const requireAuth = createServerFn().handler(async () => {
  const user = await getCurrentUser()
  if (!user) {
    throw redirect({ to: '/login' })
  }
  return user
})
```

Redirects work from:
- Server functions
- Route `beforeLoad`
- Route `loader` (via server function)

## Error Handling

Server functions can throw errors that propagate to the client:

```typescript
export const deletePost = createServerFn({ method: 'POST' })
  .inputValidator(z.object({ id: z.string() }))
  .handler(async ({ data }) => {
    const post = await db.posts.findUnique({ where: { id: data.id } })
    if (!post) {
      throw new Error('Post not found')
    }
    await db.posts.delete({ where: { id: data.id } })
    return { success: true }
  })
```

## With Middleware

```typescript
export const getProtectedData = createServerFn({ method: 'GET' })
  .middleware([authMiddleware, loggingMiddleware])
  .handler(async ({ context }) => {
    // context.user from authMiddleware
    return db.data.findMany({ where: { userId: context.user.id } })
  })
```

## Best Practices

1. **Always use `createServerFn` for server-only logic in loaders** — loaders run on both server and client
2. **Use `method: 'POST'` for mutations** — GET for reads, POST for writes
3. **Validate input with Zod** — provides runtime validation and type inference
4. **Keep server functions thin** — extract business logic into separate modules
5. **Use middleware for cross-cutting concerns** — auth, logging, rate limiting
6. **Throw `redirect()` for auth redirects** — don't return redirect objects
7. **Export server functions from dedicated files** — e.g., `src/server/posts.ts` for post-related functions
