# TanStack Start — Databases & Hosting Reference

## Database Integration

TanStack Start works with any database through server functions. All database access must be behind `createServerFn` to stay server-only.

### Pattern: Prisma

```typescript
// src/lib/db.ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const db = globalForPrisma.prisma || new PrismaClient()

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db
```

```typescript
// src/server/posts.ts
import { createServerFn } from '@tanstack/react-start'
import { db } from '~/lib/db'

export const getPosts = createServerFn().handler(async () => {
  return db.post.findMany({ orderBy: { createdAt: 'desc' } })
})

export const createPost = createServerFn({ method: 'POST' })
  .inputValidator(z.object({ title: z.string(), body: z.string() }))
  .handler(async ({ data }) => {
    return db.post.create({ data })
  })
```

### Pattern: Drizzle ORM

```typescript
// src/lib/db.ts
import { drizzle } from 'drizzle-orm/node-postgres'

export const db = drizzle(process.env.DATABASE_URL!)
```

```typescript
// src/server/users.ts
import { createServerFn } from '@tanstack/react-start'
import { db } from '~/lib/db'
import { users } from '~/lib/schema'
import { eq } from 'drizzle-orm'

export const getUser = createServerFn({ method: 'GET' })
  .inputValidator(z.object({ id: z.string() }))
  .handler(async ({ data }) => {
    const result = await db.select().from(users).where(eq(users.id, data.id))
    return result[0] ?? null
  })
```

### Critical Rules

1. **Never import database clients in route files** — use `importProtection` to block this:
   ```typescript
   tanstackStart({
     importProtection: {
       client: { specifiers: ['@prisma/client', 'drizzle-orm'] },
     },
   })
   ```
2. **Always wrap DB calls in `createServerFn`** — loaders run on both server and client
3. **Use connection pooling in production** — singleton pattern for PrismaClient, pgpool for raw pg
4. **Store connection strings in env vars** — `process.env.DATABASE_URL`

## Hosting & Deployment

### Node.js (Default)

Build and run as a Node.js server:

```bash
npm run build
node .output/server/index.mjs
```

The server listens on port 3000 by default. Override with `PORT` env var.

### Docker

```dockerfile
FROM node:22-slim AS base
WORKDIR /app

FROM base AS deps
COPY package*.json ./
RUN npm ci

FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM base AS runtime
COPY --from=build /app/.output ./.output
COPY --from=build /app/node_modules ./node_modules
ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", ".output/server/index.mjs"]
```

### Cloudflare Pages/Workers

TanStack Start supports Cloudflare deployment. Configure the adapter in your deployment settings. The Cloudflare Vite plugin can be used for local development:

```typescript
// vite.config.ts (Cloudflare-specific)
import { cloudflare } from '@cloudflare/vite-plugin'

export default defineConfig({
  plugins: [
    tsConfigPaths(),
    tanstackStart(),
    viteReact(),
    cloudflare(),
  ],
})
```

### Vercel

Deploy with Vercel's automatic framework detection or configure manually:

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel
```

### Netlify

```bash
npm i -g netlify-cli
netlify deploy
```

### Static Export

For static hosting (GitHub Pages, S3), enable full prerendering:

```typescript
tanstackStart({
  prerender: {
    enabled: true,
    routes: ['/**'],
    crawlLinks: true,
  },
})
```

Then deploy the `.output/public` directory.

### Environment Variables in Production

Set these in your hosting platform:
- `SESSION_SECRET` — At least 32 characters for session encryption
- `DATABASE_URL` — Database connection string
- `NODE_ENV=production` — Enables production optimizations
- Any `VITE_*` variables needed on the client (must be set at build time)

**Important:** `VITE_*` variables are embedded at build time, not runtime. Non-VITE variables are available at runtime via `process.env`.
