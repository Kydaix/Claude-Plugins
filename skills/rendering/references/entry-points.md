# TanStack Start — Entry Points Reference

## Server Entry Point (src/server.ts)

The server entry configures how the application is served. It creates the request handler for SSR and server routes.

### Basic Server Entry

```tsx
// src/server.ts
import handler, { createServerEntry } from '@tanstack/react-start/server-entry'

export default createServerEntry({
  fetch(request) {
    return handler.fetch(request)
  },
})
```

### Custom Server Handler

Use `defineHandlerCallback` for custom logic (logging, headers, auth):

```tsx
// src/server.ts
import {
  createStartHandler,
  defaultStreamHandler,
  defineHandlerCallback,
} from '@tanstack/react-start/server'
import { createServerEntry } from '@tanstack/react-start/server-entry'

const customHandler = defineHandlerCallback((ctx) => {
  // Custom logic: logging, headers, auth, etc.
  console.log(`Handling request: ${ctx.request.url}`)
  return defaultStreamHandler(ctx)
})

const fetch = createStartHandler(customHandler)

export default createServerEntry({ fetch })
```

### CDN Asset URL Transformation

Rewrite asset URLs to point to a CDN. Useful when the CDN URL is determined at deploy time.

#### String Prefix

```tsx
const handler = createStartHandler({
  handler: defaultStreamHandler,
  transformAssetUrls: process.env.CDN_ORIGIN || '',
})

export default createServerEntry({ fetch: handler })
```

#### Callback Function

Fine-grained control per asset type:

```tsx
const handler = createStartHandler({
  handler: defaultStreamHandler,
  transformAssetUrls: ({ url, type }) => {
    // type: 'clientEntry' | 'modulePreload' | 'stylesheet'
    if (type === 'clientEntry') return url  // Don't rewrite client entry
    return `https://cdn.example.com${url}`
  },
})
```

Assets managed by `transformAssetUrls`:
- `<link rel="modulepreload">` tags (JS preloads)
- `<link rel="stylesheet">` tags (CSS)
- Client entry `<script>` tag

#### Vite Configuration for CDN

For lazy-loaded chunks (client-side navigation), set `base: ''`:

```typescript
// vite.config.ts
export default defineConfig({
  base: '',  // Relative paths for chunks
})
```

For component-imported assets (`import logo from './logo.png'`):

```typescript
export default defineConfig({
  experimental: {
    renderBuiltUrl(filename, { hostType }) {
      if (hostType === 'js') return { relative: true }
      return `https://cdn.example.com/${filename}`
    },
  },
})
```

**Note**: `transformAssetUrls` is experimental and subject to change.

## Client Entry Point (src/client.tsx)

The client entry hydrates the server-rendered HTML on the client.

### Basic Client Entry

```tsx
// src/client.tsx
import { StartClient } from '@tanstack/react-start/client'
import { StrictMode } from 'react'
import { hydrateRoot } from 'react-dom/client'

hydrateRoot(
  document,
  <StrictMode>
    <StartClient />
  </StrictMode>,
)
```

Key points:
- `hydrateRoot(document, ...)` — hydrates the entire document
- `StartClient` — TanStack Start's client wrapper (handles router, etc.)
- `StrictMode` — React strict mode (recommended)

## Shell Component (Root Route)

The `shellComponent` on the root route defines the HTML document structure. It always renders on the server, even when `ssr: false`.

### Basic Shell

```tsx
// src/routes/__root.tsx
import { createRootRoute, HeadContent, Scripts } from '@tanstack/react-router'

export const Route = createRootRoute({
  shellComponent: RootShell,
  component: RootComponent,
})

function RootShell({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head>
        <HeadContent />
      </head>
      <body>
        {children}
        <Scripts />
      </body>
    </html>
  )
}
```

### Shell with Global Navigation

```tsx
function RootShell({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <head>
        <HeadContent />
      </head>
      <body>
        <nav>
          <Link to="/">Home</Link>
          <Link to="/about">About</Link>
        </nav>
        <main>{children}</main>
        <Scripts />
      </body>
    </html>
  )
}
```

### Shell vs Component

- **`shellComponent`** — The HTML skeleton (`<html>`, `<head>`, `<body>`). **Always renders on server.**
- **`component`** — The root component content inside the shell. Respects `ssr` setting.

When `ssr: false` on root:

```tsx
export const Route = createRootRoute({
  ssr: false,
  shellComponent: RootShell,    // Renders on server (HTML skeleton)
  component: RootComponent,     // Renders on client only
})
```

## HeadContent

`<HeadContent />` renders all `<head>` content from the current route tree:
- Meta tags from `head()` on all matched routes
- Link tags (stylesheets, icons, etc.)
- Script tags

Head content from child routes merges with parent routes. Child tags take precedence for duplicate keys.

```tsx
// Root: global meta
export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
    ],
    links: [
      { rel: 'stylesheet', href: appCss },
      { rel: 'icon', href: '/favicon.ico' },
    ],
  }),
})

// Child: page-specific meta (merges with root)
export const Route = createFileRoute('/about')({
  head: () => ({
    meta: [
      { title: 'About Us' },
      { name: 'description', content: 'Learn about our company' },
    ],
  }),
})
```

## Scripts

`<Scripts />` renders the hydration and routing scripts at the bottom of `<body>`:

```tsx
function RootShell({ children }) {
  return (
    <html>
      <head><HeadContent /></head>
      <body>
        {children}
        <Scripts />  {/* Must be at the end of body */}
      </body>
    </html>
  )
}
```

Always place `<Scripts />` at the end of `<body>` to avoid blocking rendering.

## Outlet

`<Outlet />` renders the matched child route:

```tsx
import { Outlet } from '@tanstack/react-router'

function RootComponent() {
  return (
    <div>
      <nav>...</nav>
      <main>
        <Outlet />  {/* Child route renders here */}
      </main>
      <footer>...</footer>
    </div>
  )
}
```
