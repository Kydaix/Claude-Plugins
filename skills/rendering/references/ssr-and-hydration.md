# TanStack Start — SSR & Hydration Reference

## Server-Side Rendering (SSR)

By default, TanStack Start fully server-renders every route: loaders run on the server, components render to HTML, and the client hydrates the result.

## Selective SSR

Each route can independently control its SSR behavior via the `ssr` option.

### ssr: true (default)

Full SSR — `beforeLoad`, `loader`, and component all run on the server:

```tsx
export const Route = createFileRoute('/posts')({
  // ssr: true is the default, no need to specify
  loader: () => fetchPosts(),  // Runs on server
  component: PostList,         // Renders on server, hydrates on client
})
```

### ssr: false

Nothing runs on the server. `beforeLoad`, `loader`, and the component execute only on the client during hydration:

```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  ssr: false,
  beforeLoad: () => {
    console.log('Executes on the client during hydration')
  },
  loader: () => {
    console.log('Executes on the client during hydration')
  },
  component: () => <div>This component is rendered on the client</div>,
})
```

Use cases:
- Components relying on browser APIs (canvas, WebGL, localStorage)
- Heavy visualizations (charts, maps)
- Third-party widgets without SSR support

### ssr: 'data-only'

Loader runs on the server (data is pre-fetched), but the component renders only on the client:

```tsx
export const Route = createFileRoute('/dashboard')({
  ssr: 'data-only',
  loader: () => fetchDashboardData(),  // Server
  component: DashboardViz,             // Client only
})
```

Use cases:
- Data needs to be available immediately, but component can't SSR (charts, heavy JS)
- Avoiding hydration mismatches for complex components

### ssr: function (Dynamic)

A function that runs on the server to decide SSR mode per-request. Stripped from client bundle:

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

export const Route = createFileRoute('/docs/$docType/$docId')({
  validateSearch: z.object({ details: z.boolean().optional() }),
  ssr: ({ params, search }) => {
    // Disable SSR for "sheet" documents
    if (params.status === 'success' && params.value.docType === 'sheet') {
      return false
    }
    // Data-only for detail views
    if (search.status === 'success' && search.value.details) {
      return 'data-only'
    }
    // Default: full SSR (return undefined or true)
  },
  beforeLoad: () => {
    console.log('Executes on the server depending on the result of ssr()')
  },
  loader: () => {
    console.log('Executes on the server depending on the result of ssr()')
  },
  component: () => <div>This component is rendered on the client</div>,
})
```

The `ssr` function receives:
- `params` — route params (wrapped in a result object with `status`)
- `search` — search params (wrapped in a result object with `status`)

Return values:
- `undefined` or `true` — full SSR
- `false` — no SSR
- `'data-only'` — only run loader on server

### Global SSR Disable

Disable SSR for the entire app via the root route:

```tsx
export const Route = createRootRoute({
  ssr: false,
  shellComponent: RootShell,  // HTML shell still renders on server
  component: RootComponent,   // This renders on client only
})
```

The `shellComponent` still renders on the server (to produce the `<html>` skeleton). Only the `component` is client-only.

## Hydration Errors

Hydration mismatches occur when the server-rendered HTML differs from the client-rendered HTML. React compares the DOM and throws warnings or errors.

### Common Causes

1. **Date/time formatting** — Server and client in different timezones
2. **Random values** — `Math.random()`, `crypto.randomUUID()`
3. **Browser-only APIs** — `window`, `localStorage`, `navigator`
4. **Third-party scripts** — Ads, analytics that modify DOM
5. **Conditional rendering** — Different logic on server vs client

### Strategy 1: ClientOnly Component

The recommended approach for rendering browser-only content:

```tsx
import { ClientOnly } from '@tanstack/react-router'

function MyComponent() {
  return (
    <div>
      <h1>Dashboard</h1>
      <ClientOnly fallback={<span>Loading...</span>}>
        <BrowserOnlyChart />
      </ClientOnly>
    </div>
  )
}
```

`ClientOnly` renders the `fallback` on the server and the children on the client after hydration.

### Strategy 2: useEffect for Client State

```tsx
function TimeDisplay() {
  const [time, setTime] = useState<string | null>(null)

  useEffect(() => {
    setTime(new Date().toLocaleString())
  }, [])

  return <span>{time ?? '—'}</span>
}
```

### Strategy 3: Data-Only SSR

Skip component rendering on server:

```tsx
export const Route = createFileRoute('/unstable')({
  ssr: 'data-only',
  component: () => <ExpensiveViz />,
})
```

### Strategy 4: Disable SSR

Last resort — disable SSR entirely for the route:

```tsx
export const Route = createFileRoute('/browser-only')({
  ssr: false,
})
```

## Error Boundaries

### Per-Route Error Component

```tsx
export const Route = createFileRoute('/posts/$postId')({
  errorComponent: ({ error, reset }) => (
    <div role="alert">
      <h2>Error loading post</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Retry</button>
    </div>
  ),
})
```

The `errorComponent` receives:
- `error` — the error object
- `reset` — function to retry/reset the error state

### Global Error Component

Set on the root route to catch all unhandled errors:

```tsx
export const Route = createRootRoute({
  errorComponent: ({ error }) => (
    <html>
      <body>
        <h1>Something went wrong</h1>
        <pre>{error.message}</pre>
      </body>
    </html>
  ),
})
```

### Not Found Components

```tsx
// Per-route
export const Route = createFileRoute('/posts/$postId')({
  notFoundComponent: () => (
    <div>
      <h2>Post not found</h2>
      <Link to="/posts">Back to posts</Link>
    </div>
  ),
})

// Global (root route)
export const Route = createRootRoute({
  notFoundComponent: () => (
    <div>
      <h1>404 — Page not found</h1>
      <Link to="/">Go home</Link>
    </div>
  ),
})
```

### Pending Components

Show loading state while data is being fetched:

```tsx
export const Route = createFileRoute('/posts/$postId')({
  pendingComponent: () => <div>Loading post...</div>,
  pendingMs: 1000,       // Wait 1s before showing pending
  pendingMinMs: 200,     // Show for at least 200ms to avoid flash
  loader: ({ params }) => fetchPost(params.postId),
  component: PostComponent,
})
```
