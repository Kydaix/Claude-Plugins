# TanStack Start — Observability Reference

## Overview

TanStack Start supports logging, error tracking, and monitoring through server functions, middleware, and isomorphic utilities.

## Sentry Integration

### Client-Side (React)

```tsx
// In app.tsx or client entry
import * as Sentry from '@sentry/react'

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.NODE_ENV,
})
```

### Server Functions

```tsx
import * as Sentry from '@sentry/node'

const serverFn = createServerFn().handler(async () => {
  try {
    return await riskyOperation()
  } catch (error) {
    Sentry.captureException(error)
    throw error
  }
})
```

## Server Function Logging

```tsx
import { createServerFn } from '@tanstack/react-start'

const getUser = createServerFn({ method: 'GET' })
  .inputValidator((id: string) => id)
  .handler(async ({ data: id }) => {
    const startTime = Date.now()

    try {
      console.log(`[SERVER] Fetching user ${id}`)
      const user = await db.users.findUnique({ where: { id } })

      if (!user) {
        console.log(`[SERVER] User ${id} not found`)
        throw new Error('User not found')
      }

      const duration = Date.now() - startTime
      console.log(`[SERVER] User ${id} fetched in ${duration}ms`)
      return user
    } catch (error) {
      const duration = Date.now() - startTime
      console.error(`[SERVER] Error fetching user ${id} after ${duration}ms`, error)
      throw error
    }
  })
```

## Request/Response Logging Middleware

```tsx
import { createMiddleware } from '@tanstack/react-start'

const requestLogger = createMiddleware().server(async ({ request, next }) => {
  const startTime = Date.now()
  const timestamp = new Date().toISOString()

  console.log(`[${timestamp}] ${request.method} ${request.url} - Starting`)

  try {
    const result = await next()
    const duration = Date.now() - startTime
    console.log(
      `[${timestamp}] ${request.method} ${request.url} - ${result.response.status} (${duration}ms)`,
    )
    return result
  } catch (error) {
    const duration = Date.now() - startTime
    console.error(
      `[${timestamp}] ${request.method} ${request.url} - Error (${duration}ms)`,
      error,
    )
    throw error
  }
})

// Apply to server routes
export const Route = createFileRoute('/api/users')({
  server: {
    middleware: [requestLogger],
    handlers: {
      GET: async () => {
        return Response.json({ users: await getUsers() })
      },
    },
  },
})
```

## Isomorphic Logger (Environment-Specific)

```tsx
// utils/logger.ts
import { createIsomorphicFn } from '@tanstack/react-start'

type LogLevel = 'debug' | 'info' | 'warn' | 'error'

const logger = createIsomorphicFn()
  .server((level: LogLevel, message: string, data?: any) => {
    const timestamp = new Date().toISOString()

    if (process.env.NODE_ENV === 'development') {
      console[level](`[${timestamp}] [${level.toUpperCase()}]`, message, data)
    } else {
      // Production: structured JSON for log aggregators
      console.log(JSON.stringify({
        timestamp, level, message, data,
        service: 'tanstack-start',
        environment: process.env.NODE_ENV,
      }))
    }
  })
  .client((level: LogLevel, message: string, data?: any) => {
    if (process.env.NODE_ENV === 'development') {
      console[level](`[CLIENT] [${level.toUpperCase()}]`, message, data)
    }
    // Production: send to analytics service
  })

export { logger }
```

## LLMO — llms.txt

Serve a `llms.txt` file (like `robots.txt` but for AI systems):

```ts
// src/routes/llms[.]txt.ts
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/llms.txt')({
  server: {
    handlers: {
      GET: async () => {
        const content = `# My App

> My App is a platform for building modern web applications.

## Documentation
- Getting Started: https://myapp.com/docs/getting-started
- API Reference: https://myapp.com/docs/api
`
        return new Response(content, {
          headers: { 'Content-Type': 'text/plain' },
        })
      },
    },
  },
})
```
