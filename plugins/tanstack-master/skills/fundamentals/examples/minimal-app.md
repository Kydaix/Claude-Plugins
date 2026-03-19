# Complete Minimal TanStack Start Application

Copy-paste ready files for a working TanStack Start app.

## package.json

```json
{
  "name": "my-tanstack-app",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "start": "node .output/server/index.mjs"
  },
  "dependencies": {
    "@tanstack/react-router": "^1.120.0",
    "@tanstack/react-start": "^1.120.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@vitejs/plugin-react": "^4.5.0",
    "typescript": "^5.8.0",
    "vite": "^6.3.0",
    "vite-tsconfig-paths": "^4.3.0"
  }
}
```

## tsconfig.json

```json
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

## vite.config.ts

```typescript
import { defineConfig } from 'vite'
import tsConfigPaths from 'vite-tsconfig-paths'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  server: { port: 3000 },
  plugins: [
    tsConfigPaths(),
    tanstackStart(),
    viteReact(),  // MUST be after tanstackStart
  ],
})
```

## src/router.tsx

```typescript
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

export function getRouter() {
  return createRouter({
    routeTree,
    scrollRestoration: true,
  })
}
```

## src/server.ts

```typescript
import handler, { createServerEntry } from '@tanstack/react-start/server-entry'

export default createServerEntry({
  fetch(request) {
    return handler.fetch(request)
  },
})
```

## src/client.tsx

```tsx
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

## src/routes/__root.tsx

```tsx
/// <reference types="vite/client" />
import {
  HeadContent,
  Link,
  Outlet,
  Scripts,
  createRootRoute,
} from '@tanstack/react-router'

export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { title: 'My TanStack App' },
    ],
  }),
  shellComponent: RootShell,
  errorComponent: ({ error }) => <div>Error: {error.message}</div>,
  notFoundComponent: () => <div>Page not found</div>,
})

function RootShell({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <head>
        <HeadContent />
      </head>
      <body>
        <nav style={{ padding: '1rem', display: 'flex', gap: '1rem' }}>
          <Link to="/" activeProps={{ style: { fontWeight: 'bold' } }}>
            Home
          </Link>
          <Link to="/about" activeProps={{ style: { fontWeight: 'bold' } }}>
            About
          </Link>
        </nav>
        <main style={{ padding: '1rem' }}>{children}</main>
        <Scripts />
      </body>
    </html>
  )
}
```

## src/routes/index.tsx

```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/')({
  component: HomePage,
})

function HomePage() {
  return (
    <div>
      <h1>Welcome to TanStack Start</h1>
      <p>This is a minimal working application.</p>
    </div>
  )
}
```

## src/routes/about.tsx

```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/about')({
  component: AboutPage,
})

function AboutPage() {
  return (
    <div>
      <h1>About</h1>
      <p>Built with TanStack Start.</p>
    </div>
  )
}
```
