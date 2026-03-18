# TanStack Start — Configuration Reference

## Vite Plugin Options

The `tanstackStart()` plugin accepts these options:

```typescript
import { tanstackStart } from '@tanstack/react-start/plugin/vite'

tanstackStart({
  // Static prerendering
  prerender: {
    enabled: true,
    routes: ['/blog', '/blog/posts/*'],
    crawlLinks: true,
  },

  // SPA mode (no server)
  spa: true,
  // or with options:
  spa: {
    prerender: {
      outputPath: '/custom-shell',
      crawlLinks: true,
      retryCount: 3,
    },
  },

  // Import protection
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

## Import Protection

Import protection prevents dangerous imports from leaking between environments.

### Default Behavior

TanStack Start automatically prevents certain imports. Custom rules extend (not replace) the defaults.

### Client Protection

Block server-only packages and files from the client bundle:

```typescript
importProtection: {
  client: {
    // Block specific npm packages
    specifiers: ['@prisma/client', 'bcrypt', 'nodemailer'],
    // Block files matching glob patterns
    files: ['**/db/**', '**/server/**'],
  },
}
```

### Server Protection

Block browser-only packages from the server bundle:

```typescript
importProtection: {
  server: {
    specifiers: ['localforage', 'dom-confetti'],
  },
}
```

## Environment Variables

### Server Variables

All `process.env` variables are available in:
- Server functions (`createServerFn`)
- Server-only functions (`createServerOnlyFn`)
- Server entry (`src/server.ts`)
- Middleware (server handlers)

```typescript
const getConfig = createServerFn().handler(() => {
  return {
    dbUrl: process.env.DATABASE_URL,
    apiKey: process.env.API_KEY,
    secret: process.env.SESSION_SECRET,
  }
})
```

### Client Variables

Only `VITE_`-prefixed variables are available on the client:

```
# .env
VITE_APP_NAME=My App        # ✅ Available on client
VITE_PUBLIC_API=https://...  # ✅ Available on client
DATABASE_URL=postgres://...  # ❌ Server only
SESSION_SECRET=abc123        # ❌ Server only
```

```tsx
// In components
const appName = import.meta.env.VITE_APP_NAME
```

### Critical Rules

1. **Never use `process.env` in loaders** — loaders run on both server and client
2. **Never use `process.env` in components** — components run on client
3. **Always wrap server-only env access in `createServerFn`**
4. **Only expose non-sensitive values with `VITE_` prefix**

## Path Aliases

### tsconfig.json

```json
{
  "compilerOptions": {
    "paths": {
      "~/*": ["./src/*"]
    }
  }
}
```

### Usage

```typescript
import { Button } from '~/components/Button'
import { db } from '~/lib/db'
```

The `vite-tsconfig-paths` plugin (already in the Vite config) resolves these automatically.

## Tailwind CSS Integration

### Installation

```shell
npm i -D tailwindcss @tailwindcss/vite
```

### Vite Plugin

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

### CSS File

```css
/* src/styles/app.css */
@import 'tailwindcss';
```

### Link in Root Route

```tsx
import appCss from '~/styles/app.css?url'

export const Route = createRootRoute({
  head: () => ({
    links: [{ rel: 'stylesheet', href: appCss }],
  }),
})
```

## SEO Configuration

### Per-Route Meta Tags

```tsx
export const Route = createFileRoute('/')({
  head: () => ({
    meta: [
      { title: 'My App - Home' },
      { name: 'description', content: 'Welcome to My App' },
      { property: 'og:title', content: 'My App' },
      { property: 'og:description', content: 'Welcome' },
      { property: 'og:type', content: 'website' },
    ],
    links: [
      { rel: 'canonical', href: 'https://myapp.com/' },
    ],
  }),
})
```

### Dynamic Meta from Loader Data

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => fetchPost(params.postId),
  head: ({ loaderData }) => ({
    meta: [
      { title: loaderData.title },
      { name: 'description', content: loaderData.excerpt },
    ],
  }),
})
```

### Static Prerendering for SEO

Enable prerendering for pages that need to be indexed by search engines:

```typescript
// vite.config.ts
tanstackStart({
  prerender: {
    enabled: true,
    crawlLinks: true,  // Follow links to discover routes
  },
})
```

## SPA Mode

SPA mode removes the server completely — the app runs entirely in the browser.

### Basic SPA

```typescript
tanstackStart({ spa: true })
```

### SPA with Custom Shell Prerendering

```typescript
tanstackStart({
  spa: {
    prerender: {
      outputPath: '/custom-shell',
      crawlLinks: true,
      retryCount: 3,
    },
  },
})
```

### When to Use SPA Mode

- Static hosting (GitHub Pages, S3, Netlify static)
- Apps without server-side data requirements
- Internal tools and dashboards
- Offline-first applications

### Limitations

- No SSR (no server-rendered HTML)
- No server functions
- No server routes
- SEO relies entirely on client-side rendering

## Hosting & Deployment

TanStack Start deploys to any platform that supports Node.js servers:

### Node.js Server

Default deployment — build and run:

```shell
npm run build
node .output/server/index.mjs
```

### Platform Adapters

TanStack Start supports various hosting platforms via Nitro/Vinxi presets. Configure in `vite.config.ts`:

```typescript
tanstackStart({
  // Platform-specific options
})
```

### Static Export

For static hosting, enable prerendering for all routes:

```typescript
tanstackStart({
  prerender: {
    enabled: true,
    routes: ['/**'],  // All routes
    crawlLinks: true,
  },
})
```

## DevTools

TanStack Router includes dev tools for debugging:

```tsx
import { TanStackRouterDevtools } from '@tanstack/react-router-devtools'

// In root route component
function RootComponent() {
  return (
    <>
      <Outlet />
      <TanStackRouterDevtools position="bottom-right" />
    </>
  )
}
```

Only include in development — tree-shaked in production builds.
