---
name: scaffold
description: >-
  Scaffold a new TanStack Start project with all required files configured.
  Generates vite.config.ts, tsconfig.json, router, server/client entry points,
  root route, and optionally Tailwind CSS.
argument-hint: "[project-name] [--tailwind]"
allowed-tools: ["Write", "Bash", "Read"]
---

# Scaffold a TanStack Start Project

Generate a complete, ready-to-run TanStack Start project.

## Process

1. Determine the project name from the argument (default: `my-tanstack-app`)
2. Check if `--tailwind` flag is present
3. Create the project directory structure
4. Generate all required files

## Required Files

Create these files in order:

### 1. package.json

```json
{
  "name": "PROJECT_NAME",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "start": "node .output/server/index.mjs"
  },
  "dependencies": {
    "@tanstack/react-router": "latest",
    "@tanstack/react-start": "latest",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@vitejs/plugin-react": "latest",
    "typescript": "^5.8.0",
    "vite": "latest",
    "vite-tsconfig-paths": "latest"
  }
}
```

If `--tailwind`, add to devDependencies:
```json
"tailwindcss": "latest",
"@tailwindcss/vite": "latest"
```

### 2. tsconfig.json

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
    "paths": { "~/*": ["./src/*"] }
  },
  "include": ["src"]
}
```

### 3. vite.config.ts

**CRITICAL: Plugin order must be tsConfigPaths → tanstackStart → viteReact.**

If `--tailwind`, add `tailwindcss()` after `viteReact()`.

### 4. src/router.tsx

Create with `createRouter`, `routeTree`, `scrollRestoration: true`.

### 5. src/server.ts

Basic server entry with `createServerEntry`.

### 6. src/client.tsx

Client entry with `hydrateRoot`, `StartClient`, `StrictMode`.

### 7. src/routes/__root.tsx

Root route with:
- `shellComponent` (html/head/body with HeadContent + Scripts)
- `head()` with charset, viewport, title
- `errorComponent` and `notFoundComponent`
- Navigation with `<Link>` components

If `--tailwind`, add stylesheet link to head.

### 8. src/routes/index.tsx

Simple home page component.

### 9. (If --tailwind) src/styles/app.css

```css
@import 'tailwindcss';
```

## After Generation

Run these commands:
```bash
cd PROJECT_NAME
npm install
npm run dev
```

Inform the user the app will be available at http://localhost:3000.
