# TanStack Start — Environment Variables & Next.js Migration Reference

## Environment Variable Validation with Zod

```typescript
import { z } from 'zod'

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  NODE_ENV: z.enum(['development', 'production', 'test']),
})

const clientEnvSchema = z.object({
  VITE_APP_NAME: z.string(),
  VITE_API_URL: z.string().url(),
})

// Validate at startup
export const serverEnv = envSchema.parse(process.env)
export const clientEnv = clientEnvSchema.parse(import.meta.env)
```

## TypeScript Type Declarations for Env Vars

```typescript
// env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_APP_NAME: string
  readonly VITE_API_URL: string
  readonly VITE_SENTRY_DSN?: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      readonly DATABASE_URL: string
      readonly JWT_SECRET: string
      readonly SESSION_SECRET: string
      readonly NODE_ENV: 'development' | 'production' | 'test'
    }
  }
}

export {}
```

## Required Env Validation (Simple)

```typescript
const requiredServerEnv = ['DATABASE_URL', 'JWT_SECRET'] as const
const requiredClientEnv = ['VITE_APP_NAME', 'VITE_API_URL'] as const

for (const key of requiredServerEnv) {
  if (!process.env[key]) throw new Error(`Missing required env: ${key}`)
}
for (const key of requiredClientEnv) {
  if (!import.meta.env[key]) throw new Error(`Missing required env: ${key}`)
}
```

---

## Migrating from Next.js to TanStack Start

### layout.tsx → __root.tsx

```tsx
// Next.js layout.tsx → TanStack Start __root.tsx
import { Outlet, createRootRoute, HeadContent, Scripts } from "@tanstack/react-router"
import appCss from "./globals.css?url"

export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: "utf-8" },
      { name: "viewport", content: "width=device-width, initial-scale=1" },
      { title: "My App" },
    ],
    links: [{ rel: 'stylesheet', href: appCss }],
  }),
  component: RootLayout,
})

function RootLayout() {
  return (
    <html lang="en">
      <head><HeadContent /></head>
      <body>
        <Outlet />
        <Scripts />
      </body>
    </html>
  )
}
```

### API Routes → Server Routes

```tsx
// Next.js: app/api/hello/route.ts
// export async function GET() { return Response.json("Hello") }

// TanStack Start:
export const Route = createFileRoute('/api/hello')({
  server: {
    handlers: {
      GET: async () => Response.json("Hello, World!"),
    },
  },
})
```

### getServerSideProps / Server Components → Loaders

```tsx
// Next.js:
// export async function getServerSideProps() { return { props: { posts } } }

// TanStack Start:
export const Route = createFileRoute('/')({
  loader: async () => {
    const res = await fetch('https://api.example.com/posts')
    return res.json()
  },
  component: Page,
})

function Page() {
  const posts = Route.useLoaderData()
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

### Dynamic Route Params

```tsx
// Next.js: app/posts/[slug]/page.tsx → params.slug
// TanStack Start: routes/posts/$slug.tsx → Route.useParams()

export const Route = createFileRoute('/posts/$slug')({
  component: Page,
})

function Page() {
  const { slug } = Route.useParams()
  return <div>Post: {slug}</div>
}
```

### Migration Checklist

1. Replace `layout.tsx` with `__root.tsx` using `createRootRoute`
2. Replace `page.tsx` files with route files using `createFileRoute`
3. Replace `[param]` with `$param` in file names
4. Replace API routes with `server.handlers`
5. Replace `getServerSideProps` with `loader`
6. Replace `useRouter` from `next/router` with `useNavigate` from `@tanstack/react-router`
7. Replace `<Link href="">` with `<Link to="">`
8. Replace `next/head` with `head()` on routes
9. Replace `Metadata` export with `head()` return
10. Move server-only logic into `createServerFn`
