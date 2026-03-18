---
name: TanStack Start Server
description: >-
  This skill should be used when the user asks to "create a server function",
  "add an API endpoint", "create a server route", "add middleware",
  "implement authentication", "handle form submission", "create an API",
  "add input validation", "set up sessions", "manage cookies",
  "protect a route", "add auth", "create a database query",
  "handle redirects", "create middleware chain",
  "use createServerFn", "use createMiddleware",
  or needs guidance on server-side logic, execution model, data fetching,
  authentication, middleware, or server routes in TanStack Start.
---

# TanStack Start Server

Server-side APIs in TanStack Start: server functions, middleware, server routes, execution boundaries, authentication, and sessions.

## Execution Model — Critical Concepts

TanStack Start provides four execution boundary functions:

| Function | Runs On | Callable From | Use Case |
|----------|---------|---------------|----------|
| `createServerFn` | Server | Client + Server (RPC) | Data fetching, mutations, auth |
| `createServerOnlyFn` | Server | Server only (crashes on client) | Utility functions, env access |
| `createClientOnlyFn` | Client | Client only (crashes on server) | localStorage, DOM APIs |
| `createIsomorphicFn` | Both | Both (different implementations) | Logging, analytics |

```tsx
import {
  createServerFn,
  createServerOnlyFn,
  createClientOnlyFn,
  createIsomorphicFn,
} from '@tanstack/react-start'
```

### CRITICAL: Loaders Run on Both Server and Client

```tsx
// ❌ WRONG — secret exposed to client bundle
export const Route = createFileRoute('/users')({
  loader: () => {
    const secret = process.env.SECRET  // EXPOSED!
    return fetch(`/api/users?key=${secret}`)
  },
})

// ✅ CORRECT — wrap in server function
const getUsers = createServerFn().handler(() => {
  const secret = process.env.SECRET  // Server-only, safe
  return fetch(`/api/users?key=${secret}`)
})

export const Route = createFileRoute('/users')({
  loader: () => getUsers(),
})
```

## Server Functions (createServerFn)

The primary way to run server-side code. Server functions are RPC calls — the client calls them, they execute on the server.

### Basic Server Function

```tsx
import { createServerFn } from '@tanstack/react-start'

export const getServerTime = createServerFn().handler(async () => {
  return new Date().toISOString()
})

// Call from anywhere
const time = await getServerTime()
```

### With Input Validation (Zod)

```tsx
import { z } from 'zod'

export const createUser = createServerFn({ method: 'POST' })
  .inputValidator(z.object({
    name: z.string().min(1),
    email: z.string().email(),
  }))
  .handler(async ({ data }) => {
    // data is typed: { name: string; email: string }
    return db.users.create({ data })
  })
```

### With Middleware

```tsx
export const getTodos = createServerFn({ method: 'GET' })
  .inputValidator(z.object({ userId: z.string() }))
  .middleware([authMiddleware])
  .handler(async ({ data, context }) => {
    // context.user from middleware
    return db.todos.findMany({ where: { userId: data.userId } })
  })
```

### Calling Patterns

```tsx
// From a loader
export const Route = createFileRoute('/posts')({
  loader: () => getPosts(),
})

// From a component with useServerFn + React Query
import { useQuery } from '@tanstack/react-query'

function PostList() {
  const getPosts = useServerFn(getServerPosts)
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: () => getPosts(),
  })
}
```

### Redirects from Server Functions

```tsx
import { redirect } from '@tanstack/react-router'

export const requireAuth = createServerFn().handler(async () => {
  const user = await getCurrentUser()
  if (!user) {
    throw redirect({ to: '/login' })
  }
  return user
})
```

## Middleware

Middleware customizes behavior of server routes and server functions. It is composable and hierarchical.

### Two Types

- **Request middleware** (`createMiddleware()`) — applies to all server requests (SSR, server routes, server functions)
- **Function middleware** (`createMiddleware({ type: 'function' })`) — applies to server functions only, with client + server handlers

### Basic Request Middleware

```typescript
import { createMiddleware } from '@tanstack/react-start'

const loggingMiddleware = createMiddleware().server(({ next }) => {
  console.log('Request received')
  return next()
})
```

### Function Middleware (Client + Server)

```tsx
const authMiddleware = createMiddleware({ type: 'function' })
  .client(async ({ next }) => {
    return next({
      headers: { Authorization: `Bearer ${getToken()}` },
    })
  })
  .server(async ({ next }) => {
    return next({ context: { user: await getUser() } })
  })
```

### Composing Middleware

```tsx
const loggingMiddleware = createMiddleware({ type: 'function' })
  .middleware([authMiddleware])  // depends on authMiddleware
  .server(async ({ next, context }) => {
    console.log('User:', context.user)  // from authMiddleware
    return next()
  })
```

### Context Passing

Call `next()` with a `context` object to pass data down the chain:

```tsx
const middleware = createMiddleware({ type: 'function' })
  .server(({ next }) => {
    return next({
      context: { isAwesome: Math.random() > 0.5 },
    })
  })
```

## Server Routes (API Endpoints)

Define HTTP endpoints using `server.handlers`:

```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/api/hello')({
  server: {
    handlers: {
      GET: async ({ request }) => {
        return Response.json({ message: 'Hello!' })
      },
      POST: async ({ request }) => {
        const body = await request.json()
        return new Response(`Hello, ${body.name}!`)
      },
    },
  },
})
```

Server routes follow the Web `Request`/`Response` API. They can coexist with a component in the same route file.

## Additional Resources

### Reference Files

For detailed patterns and complete examples, consult:
- **`references/server-functions.md`** — Full createServerFn API, methods, validation, error handling, advanced patterns
- **`references/middleware.md`** — Complete middleware guide: types, composing, context, use cases
- **`references/execution-model.md`** — Code execution patterns, boundaries, createServerOnlyFn, createClientOnlyFn, createIsomorphicFn
- **`references/auth-and-sessions.md`** — Authentication patterns, useSession, login/logout, route protection, session cookies
