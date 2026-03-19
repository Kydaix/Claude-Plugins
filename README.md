# TanStack-Master

Claude Code plugin providing complete TanStack Start documentation and best practices for building full-stack React applications without errors.

## Features

### Skills (Auto-activated)

| Skill | Triggers On |
|-------|------------|
| **Fundamentals** | Project setup, routing, configuration, Vite, Tailwind, SEO, environment variables, databases, hosting, SPA mode |
| **Server** | Server functions, middleware, server routes, execution model, authentication, sessions, CRUD, API endpoints |
| **Rendering** | SSR, selective SSR, hydration, static prerendering, ISR, error boundaries, entry points, loading states |

### Slash Command

| Command | Purpose |
|---------|---------|
| `/tanstack-master:scaffold` | Scaffold a new TanStack Start project with all required files |

### Agent

| Agent | Purpose |
|-------|---------|
| **tanstack-reviewer** | Reviews TanStack Start code for 25 anti-patterns across 4 severity levels |

## Installation

### Via Marketplace (recommended)

```bash
# 1. Add the marketplace
/plugin marketplace add Kydaix/Claude-Plugins

# 2. Install the plugin
/plugin install tanstack-master@kydaix-plugins

# 3. Reload plugins
/reload-plugins
```

### Local Development

```bash
claude --plugin-dir /path/to/Claude-Plugins/plugins/tanstack-master
```

## What's Covered

### Documentation (~18,000 words)

- Project setup and configuration (vite.config.ts, tsconfig, dependencies)
- File-based routing (createFileRoute, createRootRoute, dynamic routes, layouts)
- Server functions (createServerFn, inputValidator, middleware, handler)
- Execution model (createServerOnlyFn, createClientOnlyFn, createIsomorphicFn)
- Middleware (createMiddleware, types, composing, context passing)
- Server routes (HTTP handlers, Request/Response API, dynamic params)
- Authentication and sessions (useSession, login/logout, route protection)
- SSR and hydration (selective SSR, ClientOnly, hydration error strategies)
- Static prerendering and ISR (crawlLinks, on-demand revalidation)
- Entry points (server entry, client entry, shellComponent, HeadContent, Scripts)
- Environment variables and import protection
- CDN asset URL transformation
- SEO (head(), meta tags, prerendering)
- SPA mode
- Tailwind CSS and path aliases
- Database integration (Prisma, Drizzle ORM)
- Hosting and deployment (Node.js, Docker, Cloudflare, Vercel, Netlify, static export)

### Complete Examples

- Minimal app (all files, copy-paste ready)
- Authentication flow (session, middleware, protected routes, login form)
- CRUD API (Zod validation, server functions, React components, REST endpoints)
- 7 SSR/rendering patterns (full SSR, client-only, data-only, ClientOnly, dynamic SSR, ISR, SPA shell)

### Anti-Pattern Detection (25 patterns)

The reviewer agent checks for 25 anti-patterns across 4 severity levels:
- **Critical** (5): Secrets in loaders, browser APIs during SSR, wrong plugin order, insecure sessions, weak secrets
- **High** (5): GET for mutations, missing validation, editing routeTree.gen.ts, missing next(), wrong redirect
- **Medium** (10): Missing error components, no meta tags, no shellComponent, direct DB imports, no import protection, inline server logic, missing type registration, wrong env prefix, no loading states, no scroll restoration
- **Low** (5): Unnecessary SSR disable, missing DevTools, hardcoded paths, no cache config, fetch instead of server functions

## Source

Documentation sourced from [TanStack Start official docs](https://tanstack.com/start/latest/docs/framework/react/overview).
