---
name: TanStack Start Fundamentals
description: >-
  This skill should be used when the user asks to "create a TanStack Start project",
  "set up TanStack Start", "configure routing", "add a route", "create a page",
  "set up Tailwind with TanStack Start", "configure environment variables",
  "set up path aliases", "configure SEO", "deploy TanStack Start", "host TanStack Start",
  "configure SPA mode", "set up file-based routing", "configure vite for TanStack Start",
  "create a layout", "add meta tags", "configure head content",
  or needs guidance on TanStack Start project structure, configuration, routing, or setup.
  Covers project initialization, Vite configuration, file-based routing, layouts,
  navigation, environment variables, import protection, path aliases, Tailwind CSS,
  SEO, hosting, and SPA mode.
---

# TanStack Start Fundamentals

TanStack Start is a full-stack React framework powered by TanStack Router and Vite. It provides type-safe file-based routing, SSR, server functions, and a modern developer experience.

## Project Structure

A TanStack Start project follows this structure:

```
my-app/
├── src/
│   ├── routes/              # File-based routes
│   │   ├── __root.tsx       # Root layout (required)
│   │   ├── index.tsx        # Home page (/)
│   │   └── posts/
│   │       ├── index.tsx    # /posts
│   │       └── $postId.tsx  # /posts/:postId (dynamic)
│   ├── components/          # Reusable UI components
│   ├── server.ts            # Server entry point
│   ├── client.tsx           # Client entry point
│   ├── router.tsx           # Router configuration
│   └── routeTree.gen.ts     # Auto-generated route tree (DO NOT EDIT)
├── public/                  # Static assets
├── vite.config.ts           # Vite + TanStack Start plugin config
├── tsconfig.json            # TypeScript configuration
└── package.json
```

## Quick Setup

### Dependencies

```shell
npm i @tanstack/react-start @tanstack/react-router
npm i -D vite @vitejs/plugin-react vite-tsconfig-paths typescript
```

### Vite Configuration

**CRITICAL: Plugin order matters.** `tsConfigPaths` → `tanstackStart` → `viteReact`.

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import tsConfigPaths from 'vite-tsconfig-paths'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  server: { port: 3000 },
  plugins: [
    tsConfigPaths(),
    tanstackStart(),
    viteReact(),  // MUST come after tanstackStart
  ],
})
```

### Router Configuration

```typescript
// src/router.tsx
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

export function getRouter() {
  return createRouter({
    routeTree,
    scrollRestoration: true,
  })
}
```

### Root Route

Every app needs `src/routes/__root.tsx` with `createRootRoute`. It wraps all routes and defines the HTML shell via `shellComponent`.

```tsx
import { createRootRoute, HeadContent, Outlet, Scripts } from '@tanstack/react-router'

export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { title: 'My App' },
    ],
    links: [{ rel: 'stylesheet', href: '/styles.css' }],
  }),
  shellComponent: RootShell,
  errorComponent: () => <div>Error</div>,
  notFoundComponent: () => <div>Not Found</div>,
})

function RootShell({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <head><HeadContent /></head>
      <body>
        {children}
        <Scripts />
      </body>
    </html>
  )
}
```

## Routing Quick Reference

| File Path | Route | Type |
|-----------|-------|------|
| `routes/index.tsx` | `/` | Index |
| `routes/about.tsx` | `/about` | Static |
| `routes/posts/$postId.tsx` | `/posts/:postId` | Dynamic |
| `routes/posts/index.tsx` | `/posts` | Index |
| `routes/_layout.tsx` | Pathless layout | Layout |

### Defining Routes

Every route file exports a `Route` using `createFileRoute`:

```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  component: PostComponent,
  loader: async ({ params }) => fetchPost(params.postId),
  head: () => ({ meta: [{ title: 'Post' }] }),
})
```

The path string passed to `createFileRoute` is **automatically managed** by the router CLI/bundler plugin. Do not change it manually.

### Navigation

Use `<Link>` for type-safe navigation:

```tsx
import { Link } from '@tanstack/react-router'

<Link to="/posts/$postId" params={{ postId: '123' }}
  activeProps={{ className: 'font-bold' }}>
  Post 123
</Link>
```

## Environment Variables

- **Server**: Access any variable via `process.env.MY_VAR`
- **Client**: Only `VITE_`-prefixed variables via `import.meta.env.VITE_MY_VAR`
- **CRITICAL**: Never use `process.env` in client code — secrets will be exposed. Use `createServerFn` for server-only access.

## Import Protection

Configure in `vite.config.ts` to block dangerous imports per environment:

```typescript
tanstackStart({
  importProtection: {
    client: {
      specifiers: ['@prisma/client', 'bcrypt'],
      files: ['**/db/**'],
    },
    server: {
      specifiers: ['localforage'],
    },
  },
})
```

## SEO

Use `head()` on any route to set meta tags. Combine with static prerendering for best SEO.

```tsx
export const Route = createFileRoute('/')({
  head: () => ({
    meta: [
      { title: 'My App - Home' },
      { name: 'description', content: 'Welcome to My App' },
    ],
  }),
})
```

## SPA Mode

For client-only apps without a server:

```typescript
tanstackStart({ spa: true })
// or with options:
tanstackStart({
  spa: {
    prerender: { outputPath: '/custom-shell', crawlLinks: true, retryCount: 3 }
  }
})
```

## Additional Resources

### Reference Files

For detailed patterns and complete examples, consult:
- **`references/project-setup.md`** — Full project setup guide, tsconfig, dependencies, entry points (server.ts, client entry)
- **`references/routing.md`** — Complete routing guide: dynamic routes, layouts, search params, loaders, beforeLoad, head, navigation patterns
- **`references/configuration.md`** — Vite plugin options, import protection details, path aliases, Tailwind CSS integration, SPA mode details
- **`references/databases-and-hosting.md`** — Database integration (Prisma, Drizzle), hosting/deployment (Node.js, Docker, Cloudflare, Vercel, Netlify, static export)

### Examples

- **`examples/minimal-app.md`** — Complete copy-paste ready TanStack Start application (all files)
