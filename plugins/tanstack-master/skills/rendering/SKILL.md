---
name: TanStack Start Rendering
description: >-
  This skill should be used when the user asks to "configure SSR",
  "disable SSR for a route", "fix hydration errors", "set up static prerendering",
  "configure ISR", "enable incremental static regeneration",
  "use SPA mode", "add error boundaries", "handle hydration mismatch",
  "use ClientOnly", "configure selective SSR", "set up CDN assets",
  "configure the server entry point", "customize the client entry",
  "add shellComponent", "set up HeadContent and Scripts",
  "prerender pages", "configure crawlLinks",
  "configure pendingComponent", "add loading states",
  or needs guidance on rendering modes, SSR, hydration, static generation,
  error handling, loading states, or entry points in TanStack Start.
---

# TanStack Start Rendering

TanStack Start supports multiple rendering modes: full SSR (default), selective SSR, data-only SSR, static prerendering, ISR, and SPA mode. Each route can independently choose its rendering strategy.

## Rendering Modes Overview

| Mode | SSR | Data | Component | Use Case |
|------|:---:|:----:|:---------:|----------|
| Full SSR (default) | ✅ | ✅ | ✅ | SEO, fast initial load |
| `ssr: 'data-only'` | Partial | ✅ | ❌ | Complex client components |
| `ssr: false` | ❌ | ❌ | ❌ | Browser-only features |
| Static Prerendering | Build-time | ✅ | ✅ | Static content, blogs |
| ISR | Cached | ✅ | ✅ | Dynamic + cached content |
| SPA Mode | ❌ | ❌ | ❌ | Client-only apps |

## Selective SSR

Control SSR per-route with the `ssr` option:

### Disable SSR Completely

Neither `beforeLoad`, `loader`, nor the component run on the server:

```tsx
export const Route = createFileRoute('/client-app')({
  ssr: false,
  component: () => <div>Client-only</div>,
})
```

### Data-Only SSR

Loader runs on server, but component renders only on client:

```tsx
export const Route = createFileRoute('/dashboard')({
  ssr: 'data-only',
  loader: () => fetchData(),  // Runs on server
  component: DashboardViz,    // Renders on client only
})
```

### Dynamic SSR (Functional)

Decide SSR mode based on route params/search at request time. The function runs only on the server and is stripped from the client bundle:

```tsx
export const Route = createFileRoute('/docs/$docType/$docId')({
  validateSearch: z.object({ details: z.boolean().optional() }),
  // params and search are wrapped in result objects with .status and .value
  ssr: ({ params, search }) => {
    if (params.status === 'success' && params.value.docType === 'sheet') {
      return false
    }
    if (search.status === 'success' && search.value.details) {
      return 'data-only'
    }
  },
})
```

### Disable SSR for Root Route

Disable SSR globally while keeping the HTML shell:

```tsx
export const Route = createRootRoute({
  ssr: false,  // or use defaultSsr: false on the router
  shellComponent: RootShell,
  component: RootComponent,
})
```

## Hydration Errors

Hydration errors occur when server-rendered HTML doesn't match client-rendered HTML.

### Strategy 1: ClientOnly Component

```tsx
import { ClientOnly } from '@tanstack/react-router'

<ClientOnly fallback={<span>—</span>}>
  <RelativeTime ts={someTs} />
</ClientOnly>
```

### Strategy 2: useEffect for Client State

Defer client-specific values to after hydration:

```tsx
const [time, setTime] = useState<string | null>(null)
useEffect(() => { setTime(new Date().toLocaleString()) }, [])
return <span>{time ?? '—'}</span>
```

### Strategy 3: Use `ssr: 'data-only'`

Skip server rendering of the component entirely:

```tsx
export const Route = createFileRoute('/unstable')({
  ssr: 'data-only',
  component: () => <ExpensiveViz />,
})
```

### Strategy 4: Disable SSR Completely

```tsx
export const Route = createFileRoute('/browser-only')({
  ssr: false,
})
```

## Static Prerendering

Generate HTML at build time for static pages.

### Configuration

```typescript
// vite.config.ts
tanstackStart({
  prerender: {
    enabled: true,
    routes: ['/blog', '/blog/posts/*'],
    crawlLinks: true,
  },
})
```

### Full Site Prerendering

```typescript
tanstackStart({
  prerender: {
    enabled: true,
    routes: ['/**'],
    crawlLinks: true,
  },
})
```

## ISR (Incremental Static Regeneration)

Serve cached HTML and revalidate in the background. ISR combines static prerendering with route-level cache control (`staleTime`/`gcTime`) and optional on-demand revalidation.

### Vite Config

```typescript
tanstackStart({
  prerender: {
    routes: ['/blog', '/blog/posts/*'],
    crawlLinks: true,
  },
})
```

### On-Demand Revalidation

```typescript
export const Route = createFileRoute('/api/revalidate')({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const { path, secret } = await request.json()
        if (secret !== process.env.REVALIDATE_SECRET) {
          return Response.json({ error: 'Invalid token' }, { status: 401 })
        }
        // Trigger CDN purge
        await purgeCache(path)
        return Response.json({ revalidated: true })
      },
    },
  },
})
```

## Error Boundaries

### Per-Route Error Component

```tsx
export const Route = createFileRoute('/posts/$postId')({
  errorComponent: ({ error, reset }) => (
    <div>
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  ),
})
```

### Global Error Component (Root)

```tsx
export const Route = createRootRoute({
  errorComponent: ({ error }) => (
    <div>
      <h1>Application Error</h1>
      <pre>{error.message}</pre>
    </div>
  ),
})
```

### Not Found Component

```tsx
export const Route = createFileRoute('/posts/$postId')({
  notFoundComponent: () => <div>Post not found</div>,
})
```

## Additional Resources

### Reference Files

For detailed patterns and complete examples, consult:
- **`references/ssr-and-hydration.md`** — Full SSR guide, selective SSR details, hydration error strategies, ClientOnly patterns
- **`references/static-prerendering.md`** — Prerendering config, ISR setup, crawlLinks, on-demand revalidation, CDN integration
- **`references/entry-points.md`** — Server entry (createStartHandler), client entry (StartClient/hydrateRoot), shellComponent, HeadContent, Scripts, CDN asset URLs
