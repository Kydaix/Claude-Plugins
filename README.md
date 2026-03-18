# TanStack-Master

Claude Code plugin providing complete TanStack Start documentation and best practices for building full-stack React applications without errors.

## Features

### Skills (Auto-activated)

| Skill | Triggers On |
|-------|------------|
| **Fundamentals** | Project setup, routing, configuration, Vite, Tailwind, SEO, environment variables, SPA mode |
| **Server** | Server functions, middleware, server routes, execution model, authentication, sessions |
| **Rendering** | SSR, selective SSR, hydration, static prerendering, ISR, error boundaries, entry points |

### Agent

| Agent | Purpose |
|-------|---------|
| **tanstack-reviewer** | Reviews TanStack Start code for anti-patterns and best practice violations |

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

- Project setup and configuration (vite.config.ts, tsconfig, dependencies)
- File-based routing (createFileRoute, createRootRoute, dynamic routes, layouts)
- Server functions (createServerFn, inputValidator, middleware, handler)
- Execution model (createServerOnlyFn, createClientOnlyFn, createIsomorphicFn)
- Middleware (createMiddleware, types, composing, context passing)
- Server routes (HTTP handlers, Request/Response API)
- Authentication and sessions (useSession, login/logout, route protection)
- SSR and hydration (selective SSR, ClientOnly, hydration error strategies)
- Static prerendering and ISR (crawlLinks, on-demand revalidation)
- Entry points (server entry, client entry, shellComponent, HeadContent, Scripts)
- Environment variables and import protection
- CDN asset URL transformation
- SEO (head(), meta tags, prerendering)
- SPA mode
- Tailwind CSS and path aliases

## Source

Documentation sourced from [TanStack Start official docs](https://tanstack.com/start/latest/docs/framework/react/overview).
