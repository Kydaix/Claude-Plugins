# SSR & Rendering Patterns — Complete Examples

## Pattern 1: Full SSR with Data (Default)

```tsx
// src/routes/posts/$postId.tsx
import { createFileRoute } from '@tanstack/react-router'
import { createServerFn } from '@tanstack/react-start'

const fetchPost = createServerFn({ method: 'GET' })
  .inputValidator((data: { id: string }) => data)
  .handler(async ({ data }) => {
    return db.posts.findUnique({ where: { id: data.id } })
  })

export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => fetchPost({ data: { id: params.postId } }),
  head: ({ loaderData }) => ({
    meta: [
      { title: loaderData?.title ?? 'Post' },
      { name: 'description', content: loaderData?.excerpt },
      { property: 'og:title', content: loaderData?.title },
    ],
  }),
  component: PostPage,
  pendingComponent: () => <div>Loading post...</div>,
  errorComponent: ({ error, reset }) => (
    <div>
      <p>Failed to load post: {error.message}</p>
      <button onClick={reset}>Retry</button>
    </div>
  ),
  notFoundComponent: () => <div>Post not found</div>,
})

function PostPage() {
  const post = Route.useLoaderData()
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  )
}
```

## Pattern 2: Client-Only Dashboard (ssr: false)

```tsx
// src/routes/dashboard.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/dashboard')({
  ssr: false,  // Nothing runs on server
  component: DashboardPage,
})

function DashboardPage() {
  // Safe to use browser APIs directly
  const screenWidth = window.innerWidth

  return (
    <div>
      <h1>Dashboard</h1>
      <p>Screen width: {screenWidth}px</p>
      <canvas id="chart" />
    </div>
  )
}
```

## Pattern 3: Data-Only SSR (ssr: 'data-only')

```tsx
// src/routes/analytics.tsx
import { createFileRoute } from '@tanstack/react-router'
import { createServerFn } from '@tanstack/react-start'

const fetchAnalytics = createServerFn().handler(async () => {
  return db.analytics.getWeekly()
})

export const Route = createFileRoute('/analytics')({
  ssr: 'data-only',  // Loader runs on server, component on client only
  loader: () => fetchAnalytics(),
  component: AnalyticsPage,
})

function AnalyticsPage() {
  const data = Route.useLoaderData()
  // Heavy chart library that can't SSR
  return <HeavyChartLibrary data={data} />
}
```

## Pattern 4: ClientOnly for Mixed Pages

```tsx
// src/routes/index.tsx
import { createFileRoute } from '@tanstack/react-router'
import { ClientOnly } from '@tanstack/react-router'

export const Route = createFileRoute('/')({
  component: HomePage,
})

function HomePage() {
  return (
    <div>
      <h1>Welcome</h1>
      <p>This content is SSR'd normally</p>

      {/* This part only renders on client */}
      <ClientOnly fallback={<div>Loading map...</div>}>
        <InteractiveMap />
      </ClientOnly>

      {/* Relative time that would cause hydration mismatch */}
      <ClientOnly fallback={<span>—</span>}>
        <RelativeTime date={new Date()} />
      </ClientOnly>
    </div>
  )
}
```

## Pattern 5: Dynamic SSR Decision

```tsx
// src/routes/docs/$docType/$docId.tsx
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

export const Route = createFileRoute('/docs/$docType/$docId')({
  validateSearch: z.object({ preview: z.boolean().optional() }),
  // params and search are wrapped in result objects
  ssr: ({ params, search }) => {
    // Spreadsheets can't SSR
    if (params.status === 'success' && params.value.docType === 'sheet') {
      return false
    }
    // Preview mode: data only
    if (search.status === 'success' && search.value.preview) {
      return 'data-only'
    }
    // Default: full SSR
  },
  loader: ({ params }) => fetchDoc(params.docType, params.docId),
  component: DocViewer,
})
```

## Pattern 6: Static Prerendering with ISR

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import tsConfigPaths from 'vite-tsconfig-paths'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    tsConfigPaths(),
    tanstackStart({
      prerender: {
        enabled: true,
        routes: ['/', '/about', '/blog', '/blog/posts/*'],
        crawlLinks: true,
      },
    }),
    viteReact(),
  ],
})
```

```tsx
// src/routes/blog/posts/$slug.tsx — Route with cache control
export const Route = createFileRoute('/blog/posts/$slug')({
  loader: ({ params }) => fetchBlogPost(params.slug),
  staleTime: 60_000,          // Fresh for 1 minute
  gcTime: 10 * 60_000,        // Keep in cache 10 minutes
  head: ({ loaderData }) => ({
    meta: [
      { title: loaderData.title },
      { name: 'description', content: loaderData.excerpt },
    ],
  }),
  component: BlogPost,
})
```

## Pattern 7: SPA Shell with Disabled Root SSR

```tsx
// src/routes/__root.tsx
import {
  createRootRoute,
  HeadContent,
  Outlet,
  Scripts,
} from '@tanstack/react-router'

export const Route = createRootRoute({
  ssr: false,  // Disable SSR for all child routes
  shellComponent: RootShell,  // Shell still renders on server
  component: RootComponent,
})

// This always renders on server (HTML skeleton)
function RootShell({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head><HeadContent /></head>
      <body>{children}<Scripts /></body>
    </html>
  )
}

// This renders on client only
function RootComponent() {
  return (
    <div>
      <nav>...</nav>
      <Outlet />
    </div>
  )
}
```
