# TanStack Start — Server Routes (API Endpoints) Reference

## Overview

Server routes define HTTP endpoints using the `server.handlers` object on a route. They follow the standard Web `Request`/`Response` API and support all HTTP methods.

## Basic Server Route

```typescript
// src/routes/api/hello.ts
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/api/hello')({
  server: {
    handlers: {
      GET: async ({ request }) => {
        return Response.json({ message: 'Hello, World!' })
      },
    },
  },
})
```

## HTTP Methods

All standard HTTP methods are supported:

```typescript
export const Route = createFileRoute('/api/posts')({
  server: {
    handlers: {
      GET: async ({ request }) => {
        const posts = await db.posts.findMany()
        return Response.json(posts)
      },
      POST: async ({ request }) => {
        const body = await request.json()
        const post = await db.posts.create({ data: body })
        return Response.json(post, { status: 201 })
      },
      PUT: async ({ request }) => {
        const body = await request.json()
        const post = await db.posts.update({ where: { id: body.id }, data: body })
        return Response.json(post)
      },
      DELETE: async ({ request }) => {
        const url = new URL(request.url)
        const id = url.searchParams.get('id')
        await db.posts.delete({ where: { id } })
        return new Response(null, { status: 204 })
      },
    },
  },
})
```

## Request Object

The handler receives a standard `Request` object:

```typescript
POST: async ({ request }) => {
  // URL and search params
  const url = new URL(request.url)
  const page = url.searchParams.get('page')

  // Headers
  const authHeader = request.headers.get('Authorization')

  // Body parsing
  const jsonBody = await request.json()
  const textBody = await request.text()
  const formBody = await request.formData()

  // Method
  console.log(request.method)  // 'POST'
}
```

## Response Patterns

```typescript
// JSON response
return Response.json({ data: 'value' })

// JSON with status code
return Response.json({ error: 'Not found' }, { status: 404 })

// Plain text
return new Response('Hello, World!')

// No content
return new Response(null, { status: 204 })

// Custom headers
return new Response(JSON.stringify(data), {
  status: 200,
  headers: {
    'Content-Type': 'application/json',
    'Cache-Control': 'max-age=3600',
    'X-Custom-Header': 'value',
  },
})

// Redirect
return Response.redirect('https://example.com', 302)
```

## Co-located Server Route + Component

A single file can define both an API endpoint and a page:

```tsx
// src/routes/hello.tsx
import { createFileRoute } from '@tanstack/react-router'
import { useState } from 'react'

export const Route = createFileRoute('/hello')({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const body = await request.json()
        return Response.json({ message: `Hello, ${body.name}!` })
      },
    },
  },
  component: HelloPage,
})

function HelloPage() {
  const [reply, setReply] = useState('')

  return (
    <div>
      <button
        onClick={() => {
          fetch('/hello', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ name: 'World' }),
          })
            .then((res) => res.json())
            .then((data) => setReply(data.message))
        }}
      >
        Say Hello
      </button>
      {reply && <p>{reply}</p>}
    </div>
  )
}
```

## Dynamic Route Params in Server Routes

```typescript
// src/routes/api/posts/$postId.ts
export const Route = createFileRoute('/api/posts/$postId')({
  server: {
    handlers: {
      GET: async ({ request, params }) => {
        const post = await db.posts.findUnique({ where: { id: params.postId } })
        if (!post) {
          return Response.json({ error: 'Not found' }, { status: 404 })
        }
        return Response.json(post)
      },
    },
  },
})
```

## Server Routes vs Server Functions

| Feature | Server Routes | Server Functions |
|---------|--------------|-----------------|
| HTTP Methods | GET, POST, PUT, DELETE, etc. | GET or POST only |
| Type Safety | Manual (Request/Response) | Automatic (inputValidator) |
| Middleware | Not directly (use request middleware) | `.middleware([])` chain |
| Use Case | External API, webhooks, third-party integration | Internal data fetching, mutations |
| Call Pattern | `fetch('/api/...')` | Direct function call |

**Rule of thumb:** Use server functions for internal app data. Use server routes for external-facing APIs and webhooks.
