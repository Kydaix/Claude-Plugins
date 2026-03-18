# TanStack Start — Middleware Reference

## Overview

Middleware customizes the behavior of server routes and server functions. It is composable, hierarchical, and supports both request-level and function-level logic.

## Use Cases

- **Authentication** — Verify user identity before executing server functions
- **Authorization** — Check permissions
- **Logging** — Log requests, responses, errors
- **CSP** — Configure Content Security Policy headers
- **Observability** — Collect metrics, traces, logs
- **Context** — Attach data for use in downstream functions
- **Error Handling** — Handle errors consistently

## Two Middleware Types

### Request Middleware

Applies to **all** server requests: SSR, server routes, and server functions.

```typescript
import { createMiddleware } from '@tanstack/react-start'

const loggingMiddleware = createMiddleware().server(({ next }) => {
  console.log('Incoming request')
  return next()
})
```

### Function Middleware (`type: 'function'`)

Applies to **server functions only**. Has separate client and server handlers.

```tsx
const authMiddleware = createMiddleware({ type: 'function' })
  .client(async ({ next }) => {
    // Runs on the CLIENT before the RPC call
    return next({
      headers: { Authorization: `Bearer ${getToken()}` },
    })
  })
  .server(async ({ next }) => {
    // Runs on the SERVER when the RPC arrives
    const user = await getUser()
    return next({ context: { user } })
  })
```

Key differences:
- `.client()` runs on the client before the server call is made
- `.server()` runs on the server when the request arrives
- Client handler can add headers to the RPC request
- Server handler can add context for downstream middleware/handlers

## The `next()` Function

Every middleware **must** call `next()` to pass control to the next middleware or handler.

### Basic next()

```typescript
const middleware = createMiddleware().server(({ next }) => {
  // Do something
  return next()  // Pass control
})
```

### next() with Context

Merge data into the context object for downstream middleware and handlers:

```typescript
const middleware = createMiddleware({ type: 'function' })
  .server(({ next }) => {
    return next({
      context: {
        isAwesome: true,
        timestamp: Date.now(),
      },
    })
  })
```

### next() with Headers

Add headers to the outgoing request (client middleware):

```typescript
const middleware = createMiddleware({ type: 'function' })
  .client(({ next }) => {
    return next({
      headers: {
        'X-Custom-Header': 'value',
        Authorization: `Bearer ${getToken()}`,
      },
    })
  })
```

## Composing Middleware

Middleware can depend on other middleware via the `.middleware()` method:

```tsx
// Base: auth middleware
const authMiddleware = createMiddleware({ type: 'function' })
  .server(async ({ next }) => {
    const user = await validateToken()
    if (!user) throw new Error('Unauthorized')
    return next({ context: { user } })
  })

// Depends on auth: logging middleware
const loggingMiddleware = createMiddleware({ type: 'function' })
  .middleware([authMiddleware])
  .server(async ({ next, context }) => {
    console.log(`User ${context.user.id} made a request`)
    return next()
  })

// Use in server function
const getProtectedData = createServerFn()
  .middleware([loggingMiddleware])  // Automatically includes authMiddleware
  .handler(async ({ context }) => {
    // context.user is available (from authMiddleware)
    return db.data.findMany({ where: { userId: context.user.id } })
  })
```

The middleware chain executes in order: `authMiddleware` → `loggingMiddleware` → `handler`.

## Authentication Middleware Pattern

Complete auth middleware example:

```tsx
// middleware/auth.ts
import { createMiddleware } from '@tanstack/react-start'

export const authMiddleware = createMiddleware({ type: 'function' })
  .client(async ({ next }) => {
    const token = localStorage.getItem('auth-token')
    if (token) {
      return next({
        headers: { Authorization: `Bearer ${token}` },
      })
    }
    return next()
  })
  .server(async ({ next }) => {
    const user = await validateSession()
    if (!user) {
      throw redirect({ to: '/login' })
    }
    return next({
      context: { user },
    })
  })
```

## Admin Authorization Middleware

```tsx
export const adminMiddleware = createMiddleware({ type: 'function' })
  .middleware([authMiddleware])
  .server(async ({ next, context }) => {
    if (context.user.role !== 'admin') {
      throw new Error('Forbidden: Admin access required')
    }
    return next()
  })
```

## Request Timing Middleware

```typescript
const timingMiddleware = createMiddleware().server(async ({ next }) => {
  const start = Date.now()
  const result = await next()
  const duration = Date.now() - start
  console.log(`Request took ${duration}ms`)
  return result
})
```

## Best Practices

1. **Use `type: 'function'` for server function middleware** — it provides client + server handlers and context passing
2. **Use plain `createMiddleware()` for global request middleware** — SSR, server routes, etc.
3. **Always call `next()`** — forgetting to call next breaks the chain
4. **Pass data via `context`, not global state** — type-safe and composable
5. **Compose middleware hierarchically** — auth → authorization → logging
6. **Keep middleware focused** — one responsibility per middleware
7. **Throw errors or redirects for auth failures** — don't return error objects
