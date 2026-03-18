# TanStack Start — Static Prerendering & ISR Reference

## Static Prerendering

Static prerendering generates HTML at build time. Pages are pre-built and ready for instant delivery, significantly improving initial load performance and SEO.

### Basic Configuration

```typescript
// vite.config.ts
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [
    tanstackStart({
      prerender: {
        enabled: true,
        routes: ['/blog', '/blog/posts/*'],
        crawlLinks: true,
      },
    }),
  ],
})
```

### Options

- `enabled` — Enable/disable prerendering
- `routes` — Array of route patterns to prerender. Supports wildcards (`*`)
- `crawlLinks` — Follow `<Link>` components to discover additional routes

### Full Site Prerendering

```typescript
tanstackStart({
  prerender: {
    enabled: true,
    routes: ['/**'],      // All routes
    crawlLinks: true,     // Discover routes from links
  },
})
```

### Selective Prerendering

Only prerender specific pages:

```typescript
tanstackStart({
  prerender: {
    enabled: true,
    routes: [
      '/',              // Home
      '/about',         // About
      '/blog',          // Blog index
      '/blog/posts/*',  // All blog posts
      '/pricing',       // Pricing
    ],
    crawlLinks: false,  // Don't auto-discover
  },
})
```

### When to Prerender

- Marketing/landing pages
- Blog posts and documentation
- FAQ and help pages
- Any content that doesn't change per-user

### When NOT to Prerender

- Authenticated/personalized pages
- Real-time data (dashboards, feeds)
- User-specific content
- Pages with dynamic URL params that can't be enumerated

## Incremental Static Regeneration (ISR)

ISR combines the benefits of static generation with the ability to update content without a full rebuild.

### How ISR Works

1. Page is statically generated at build time
2. Served from cache on subsequent requests
3. Cache revalidated in the background after a configurable time
4. Or explicitly purged via an API endpoint

### ISR Configuration

```typescript
// vite.config.ts
tanstackStart({
  prerender: {
    routes: ['/blog', '/blog/posts/*'],
    crawlLinks: true,
  },
})
```

### Route-Level Cache Control

Use `staleTime` and `gcTime` on routes:

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => fetchPost(params.postId),
  staleTime: 10_000,       // Data is fresh for 10 seconds
  gcTime: 5 * 60_000,      // Keep in cache for 5 minutes
})
```

### On-Demand Revalidation

Create an API endpoint to purge cache on demand:

```typescript
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/api/revalidate')({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const { path, secret } = await request.json()

        // Verify secret token
        if (secret !== process.env.REVALIDATE_SECRET) {
          return Response.json({ error: 'Invalid token' }, { status: 401 })
        }

        // Trigger CDN purge via your CDN's API
        // Example with Cloudflare:
        await fetch(
          `https://api.cloudflare.com/client/v4/zones/${process.env.ZONE_ID}/purge_cache`,
          {
            method: 'POST',
            headers: {
              Authorization: `Bearer ${process.env.CF_API_TOKEN}`,
              'Content-Type': 'application/json',
            },
            body: JSON.stringify({
              files: [`https://yoursite.com${path}`],
            }),
          },
        )

        return Response.json({ revalidated: true })
      },
    },
  },
})
```

### Triggering Revalidation

Call the revalidation endpoint from a CMS webhook, admin panel, or cron job:

```bash
curl -X POST https://yoursite.com/api/revalidate \
  -H "Content-Type: application/json" \
  -d '{"path": "/blog/posts/my-post", "secret": "your-secret"}'
```

## SEO with Prerendering

Static prerendering is the best strategy for SEO because search engines receive fully-rendered HTML:

```typescript
// Combine prerendering with per-route meta tags
tanstackStart({
  prerender: {
    enabled: true,
    crawlLinks: true,
  },
})

// In route
export const Route = createFileRoute('/blog/$slug')({
  head: ({ loaderData }) => ({
    meta: [
      { title: loaderData.title },
      { name: 'description', content: loaderData.excerpt },
      { property: 'og:title', content: loaderData.title },
    ],
  }),
})
```

## SPA Mode

For client-only applications without a server.

### Basic SPA

```typescript
tanstackStart({ spa: true })
```

### SPA with Custom Shell

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

### SPA Limitations

- No SSR — search engines see empty HTML (poor SEO)
- No server functions (`createServerFn` unavailable)
- No server routes
- No middleware
- Data fetching must be client-side only
