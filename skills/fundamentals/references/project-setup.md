# TanStack Start — Project Setup Reference

## Installation

### From Scratch

```shell
mkdir myApp && cd myApp
npm init -y
npm i @tanstack/react-start @tanstack/react-router
npm i -D vite @vitejs/plugin-react vite-tsconfig-paths typescript
```

### Using CLI (Recommended)

```shell
npx create-tanstack-app@latest my-app
```

## Vite Configuration

**CRITICAL: Plugin order is mandatory.** `tsConfigPaths` first, then `tanstackStart`, then `viteReact`.

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import tsConfigPaths from 'vite-tsconfig-paths'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  server: {
    port: 3000,
  },
  plugins: [
    tsConfigPaths(),
    tanstackStart(),
    // React's Vite plugin MUST come after Start's Vite plugin
    viteReact(),
  ],
})
```

### TanStack Start Plugin Options

```typescript
tanstackStart({
  // Static prerendering configuration
  prerender: {
    enabled: true,
    routes: ['/blog', '/blog/posts/*'],
    crawlLinks: true,
  },
  // SPA mode (no server)
  spa: true,
  // Import protection
  importProtection: {
    client: {
      specifiers: ['@prisma/client'],
      files: ['**/db/**'],
    },
    server: {
      specifiers: ['localforage'],
    },
  },
})
```

## TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "paths": {
      "~/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

## Router Configuration

```typescript
// src/router.tsx
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

export function getRouter() {
  const router = createRouter({
    routeTree,
    scrollRestoration: true,
  })
  return router
}
```

The `routeTree.gen.ts` file is auto-generated. **Never edit it manually.**

## Server Entry Point

The server entry point configures how the app is served.

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

```tsx
// src/server.ts
import {
  createStartHandler,
  defaultStreamHandler,
  defineHandlerCallback,
} from '@tanstack/react-start/server'
import { createServerEntry } from '@tanstack/react-start/server-entry'

const customHandler = defineHandlerCallback((ctx) => {
  // Add custom logic here (logging, auth, etc.)
  return defaultStreamHandler(ctx)
})

const fetch = createStartHandler(customHandler)

export default createServerEntry({ fetch })
```

### CDN Asset URLs

For serving static assets from a CDN:

```tsx
// src/server.ts
import {
  createStartHandler,
  defaultStreamHandler,
} from '@tanstack/react-start/server'
import { createServerEntry } from '@tanstack/react-start/server-entry'

// Simple string prefix
const handler = createStartHandler({
  handler: defaultStreamHandler,
  transformAssetUrls: process.env.CDN_ORIGIN || '',
})

export default createServerEntry({ fetch: handler })
```

Callback variant for fine-grained control:

```tsx
const handler = createStartHandler({
  handler: defaultStreamHandler,
  transformAssetUrls: ({ url, type }) => {
    if (type === 'clientEntry') return url
    return `https://cdn.example.com${url}`
  },
})
```

Also set `base: ''` in `vite.config.ts` for relative chunk paths:

```typescript
export default defineConfig({
  base: '',
  // ...
})
```

And optionally use `experimental.renderBuiltUrl` for component-imported assets:

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

**Note**: `transformAssetUrls` is **experimental** and subject to change.

## Client Entry Point

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

## Root Route (Shell)

The root route (`src/routes/__root.tsx`) is the entry point for all routes. It defines:
- HTML document structure via `shellComponent`
- Global `<head>` content (meta, links, scripts)
- Error and not-found fallbacks
- Global navigation layout

```tsx
/// <reference types="vite/client" />
import {
  HeadContent,
  Link,
  Outlet,
  Scripts,
  createRootRoute,
} from '@tanstack/react-router'
import appCss from '~/styles/app.css?url'

export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { title: 'My App' },
      { name: 'description', content: 'My TanStack Start App' },
    ],
    links: [
      { rel: 'stylesheet', href: appCss },
      { rel: 'icon', href: '/favicon.ico' },
    ],
    scripts: [],
  }),
  errorComponent: ({ error }) => <div>Error: {error.message}</div>,
  notFoundComponent: () => <div>Page not found</div>,
  shellComponent: RootShell,
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

## Project Structure Summary

```
my-app/
├── src/
│   ├── routes/              # File-based routes
│   │   ├── __root.tsx       # Root layout (REQUIRED)
│   │   ├── index.tsx        # / (home page)
│   │   ├── about.tsx        # /about
│   │   └── posts/
│   │       ├── index.tsx    # /posts
│   │       └── $postId.tsx  # /posts/:postId
│   ├── components/          # Reusable components
│   ├── server.ts            # Server entry point
│   ├── client.tsx           # Client entry point
│   ├── router.tsx           # Router configuration
│   └── routeTree.gen.ts     # Auto-generated (DO NOT EDIT)
├── public/                  # Static assets
├── vite.config.ts           # Vite configuration
├── tsconfig.json            # TypeScript config
└── package.json
```

## Environment Variables

### Server-Side

Access any env variable in server functions or server entry:

```typescript
// In createServerFn or server.ts
const dbUrl = process.env.DATABASE_URL  // ✅ Server-only, safe
```

### Client-Side

Only `VITE_`-prefixed variables are available:

```tsx
// In components
const appName = import.meta.env.VITE_APP_NAME  // ✅ Client-safe
```

### CRITICAL Pitfall

**Loaders run on BOTH server and client.** Never use `process.env` directly in a loader:

```tsx
// ❌ WRONG — secret exposed to client
export const Route = createFileRoute('/users')({
  loader: () => {
    const secret = process.env.SECRET
    return fetch(`/api/users?key=${secret}`)
  },
})

// ✅ CORRECT — wrap in server function
const getUsers = createServerFn().handler(() => {
  const secret = process.env.SECRET
  return fetch(`/api/users?key=${secret}`)
})

export const Route = createFileRoute('/users')({
  loader: () => getUsers(),
})
```

## Path Aliases

Configure in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "paths": {
      "~/*": ["./src/*"]
    }
  }
}
```

Then use in imports:

```typescript
import { Button } from '~/components/Button'
```

The `vite-tsconfig-paths` plugin resolves these automatically.

## Tailwind CSS Integration

```shell
npm i -D tailwindcss @tailwindcss/vite
```

Add the Vite plugin:

```typescript
// vite.config.ts
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [
    tsConfigPaths(),
    tanstackStart(),
    viteReact(),
    tailwindcss(),
  ],
})
```

Import in your CSS:

```css
/* src/styles/app.css */
@import 'tailwindcss';
```

Link in root route:

```tsx
import appCss from '~/styles/app.css?url'

export const Route = createRootRoute({
  head: () => ({
    links: [{ rel: 'stylesheet', href: appCss }],
  }),
})
```
