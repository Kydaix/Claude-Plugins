# TanStack Start — Routing Reference

## File-Based Routing

TanStack Start uses file-based routing powered by TanStack Router. Files in `src/routes/` automatically become routes. The route tree is generated in `src/routeTree.gen.ts` — **never edit this file manually**.

## Route File Convention

Every route file must export a `Route` using `createFileRoute`:

```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/path')({
  component: MyComponent,
})
```

The path string is **automatically managed** by the router's bundler plugin. As files are created, moved, or renamed, paths update automatically.

## Route Types

### Index Routes

```tsx
// src/routes/index.tsx → /
export const Route = createFileRoute('/')({
  component: HomePage,
})
```

### Static Routes

```tsx
// src/routes/about.tsx → /about
export const Route = createFileRoute('/about')({
  component: AboutPage,
})
```

### Dynamic Routes

Use `$` prefix for dynamic segments:

```tsx
// src/routes/posts/$postId.tsx → /posts/:postId
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  component: PostPage,
})

function PostPage() {
  const { postId } = Route.useParams()
  return <div>Post: {postId}</div>
}
```

### Nested Routes

```
src/routes/
├── posts/
│   ├── index.tsx        # /posts
│   └── $postId.tsx      # /posts/:postId
```

### Layout Routes (Pathless)

Prefix with `_` for layouts that don't add a URL segment:

```
src/routes/
├── _layout.tsx          # Wraps children without adding URL segment
├── _layout/
│   ├── page-a.tsx       # /page-a
│   └── page-b.tsx       # /page-b
```

## Route Options

`createFileRoute` accepts a comprehensive options object:

```tsx
export const Route = createFileRoute('/posts/$postId')({
  // Component to render
  component: PostComponent,

  // Data loading (runs on server during SSR, then client on navigation)
  loader: async ({ params }) => {
    return fetchPost(params.postId)
  },

  // Runs before the loader
  beforeLoad: async ({ params, context }) => {
    // Auth checks, redirects, context setup
  },

  // Route-level <head> content
  head: ({ loaderData }) => ({
    meta: [
      { title: loaderData?.title ?? 'Post' },
      { name: 'description', content: loaderData?.summary },
    ],
  }),

  // Cache control
  staleTime: 10_000,       // Fresh for 10 seconds
  gcTime: 5 * 60_000,      // Keep in memory for 5 minutes

  // SSR control
  ssr: true,               // true | false | 'data-only' | function

  // Search params validation
  validateSearch: (search) => ({
    page: Number(search.page) || 1,
  }),

  // Error fallback
  errorComponent: ({ error }) => <div>Error: {error.message}</div>,

  // Not found fallback
  notFoundComponent: () => <div>Not found</div>,

  // Pending/loading fallback
  pendingComponent: () => <div>Loading...</div>,

  // Minimum pending time (ms) to avoid flash
  pendingMinMs: 200,

  // Timeout before showing pending component
  pendingMs: 1000,
})
```

## Loaders

Loaders fetch data for a route. They run during SSR on the server, then on the client during navigation.

**CRITICAL: Loaders run on BOTH server and client.** Use `createServerFn` for server-only logic.

```tsx
// ✅ Safe pattern
const fetchPost = createServerFn({ method: 'GET' })
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    return db.posts.findUnique({ where: { id: data.id } })
  })

export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => fetchPost({ data: { id: params.postId } }),
  component: PostComponent,
})

function PostComponent() {
  const post = Route.useLoaderData()
  return <h1>{post.title}</h1>
}
```

### Loader with Caching

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => fetchPost(params.postId),
  staleTime: 10_000,       // Data fresh for 10s
  gcTime: 5 * 60_000,      // Keep in cache for 5min
})
```

## beforeLoad

Runs before the loader. Use for auth checks, redirects, and setting up context.

```tsx
import { redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/dashboard')({
  beforeLoad: async () => {
    const user = await requireAuth()
    if (!user) {
      throw redirect({ to: '/login' })
    }
    return { user }
  },
  loader: async ({ context }) => {
    // context.user is available here
    return fetchDashboard(context.user.id)
  },
})
```

## Head / Meta Tags / SEO

Each route can define `<head>` content:

```tsx
export const Route = createFileRoute('/posts/$postId')({
  head: ({ loaderData }) => ({
    meta: [
      { title: loaderData.title },
      { name: 'description', content: loaderData.summary },
      // Open Graph
      { property: 'og:title', content: loaderData.title },
      { property: 'og:description', content: loaderData.summary },
      { property: 'og:type', content: 'article' },
    ],
    links: [
      { rel: 'canonical', href: `https://mysite.com/posts/${loaderData.id}` },
    ],
  }),
})
```

Head content from child routes merges with parent routes.

## Navigation

### Link Component

```tsx
import { Link } from '@tanstack/react-router'

// Static route
<Link to="/about">About</Link>

// Dynamic route with params
<Link to="/posts/$postId" params={{ postId: '123' }}>Post 123</Link>

// With search params
<Link to="/posts" search={{ page: 2 }}>Page 2</Link>

// Active styling
<Link
  to="/posts"
  activeProps={{ className: 'font-bold text-blue-600' }}
  activeOptions={{ exact: true }}
>
  Posts
</Link>
```

### Programmatic Navigation

```tsx
import { useNavigate } from '@tanstack/react-router'

function MyComponent() {
  const navigate = useNavigate()

  const handleClick = () => {
    navigate({ to: '/posts/$postId', params: { postId: '123' } })
  }
}
```

## Search Params

Validate and type search params with `validateSearch`:

```tsx
import { z } from 'zod'

export const Route = createFileRoute('/posts')({
  validateSearch: z.object({
    page: z.number().optional().default(1),
    sort: z.enum(['date', 'title']).optional().default('date'),
  }),
  component: PostList,
})

function PostList() {
  const { page, sort } = Route.useSearch()
  return <div>Page {page}, sorted by {sort}</div>
}
```

## Hooks Available in Route Components

```tsx
function MyComponent() {
  // Access route params
  const params = Route.useParams()

  // Access loader data
  const data = Route.useLoaderData()

  // Access search params
  const search = Route.useSearch()

  // Access route context
  const context = Route.useRouteContext()
}
```

## Root Route

The root route is special — use `createRootRoute` (not `createFileRoute`):

```tsx
// src/routes/__root.tsx
import { createRootRoute, Outlet } from '@tanstack/react-router'

export const Route = createRootRoute({
  component: RootComponent,
  shellComponent: RootShell,
  head: () => ({ /* global meta */ }),
  errorComponent: GlobalError,
  notFoundComponent: GlobalNotFound,
})

function RootComponent() {
  return (
    <div>
      <nav>{/* Global navigation */}</nav>
      <Outlet />
    </div>
  )
}

function RootShell({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head><HeadContent /></head>
      <body>{children}<Scripts /></body>
    </html>
  )
}
```

Key differences from `createFileRoute`:
- Uses `createRootRoute` (no path argument)
- Has `shellComponent` for the HTML document wrapper
- `<HeadContent />` renders all head content from the route tree
- `<Scripts />` renders hydration scripts
- `<Outlet />` renders child routes
