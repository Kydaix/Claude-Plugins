# TanStack Start Anti-Patterns — Exhaustive Checklist

Use this checklist when reviewing TanStack Start code. Each anti-pattern includes what's wrong, why it matters, and how to fix it.

---

## CRITICAL (Security / Crashes)

### AP-01: Secrets in Loaders

**What:** Using `process.env` directly in route loaders.
**Why:** Loaders run on BOTH server and client. Secrets leak into the client bundle.
**Detect:** Search for `process.env` inside `loader:` or `beforeLoad:` blocks.

```tsx
// ❌ BAD
loader: () => { const key = process.env.API_KEY; ... }

// ✅ FIX
const getData = createServerFn().handler(() => { const key = process.env.API_KEY; ... })
loader: () => getData()
```

### AP-02: Browser APIs During SSR

**What:** Using `window`, `document`, `localStorage`, `navigator` in components without guards.
**Why:** These don't exist on the server. The app crashes during SSR.
**Detect:** Search for `window.`, `document.`, `localStorage.`, `navigator.` outside of `useEffect` or `ClientOnly`.

```tsx
// ❌ BAD
function Component() { const w = window.innerWidth; ... }

// ✅ FIX — Option A: ClientOnly
<ClientOnly fallback={<span>—</span>}><BrowserComponent /></ClientOnly>

// ✅ FIX — Option B: useEffect
const [width, setWidth] = useState(0)
useEffect(() => { setWidth(window.innerWidth) }, [])
```

### AP-03: Wrong Vite Plugin Order

**What:** `viteReact()` placed before `tanstackStart()` in vite.config.ts.
**Why:** TanStack Start's plugin must process files before React's plugin. Wrong order causes build failures or silent bugs.
**Detect:** Read `vite.config.ts`, check plugin array order.

```typescript
// ❌ BAD
plugins: [viteReact(), tanstackStart()]

// ✅ FIX
plugins: [tsConfigPaths(), tanstackStart(), viteReact()]
```

### AP-04: Session Cookies Without Security Flags

**What:** `useSession` configured without `httpOnly`, `secure`, or `sameSite`.
**Why:** XSS can steal session cookies. CSRF attacks possible.
**Detect:** Search for `useSession` calls, check cookie config.

```typescript
// ❌ BAD
useSession({ password: secret })

// ✅ FIX
useSession({
  password: secret,
  cookie: { httpOnly: true, secure: process.env.NODE_ENV === 'production', sameSite: 'lax' }
})
```

### AP-05: Short/Hardcoded Session Secret

**What:** Session password shorter than 32 characters or hardcoded in source.
**Why:** Weak secrets allow session forgery. Hardcoded secrets leak via source control.
**Detect:** Search for `password:` near `useSession`, check length and source.

```typescript
// ❌ BAD
password: 'short'
password: 'my-super-secret-session-password-123'

// ✅ FIX
password: process.env.SESSION_SECRET!  // 32+ char random string from env
```

---

## HIGH (Correctness / Data Integrity)

### AP-06: GET for Mutations

**What:** Using default `createServerFn()` (GET) for write operations.
**Why:** GET requests can be cached, prefetched, or replayed by browsers. Mutations should use POST.
**Detect:** Search for `createServerFn()` without `{ method: 'POST' }` that contain write operations (create, update, delete).

```tsx
// ❌ BAD
const deletePost = createServerFn().handler(async () => { await db.posts.delete(...) })

// ✅ FIX
const deletePost = createServerFn({ method: 'POST' }).handler(...)
```

### AP-07: Missing Input Validation

**What:** POST server functions without `.inputValidator()`.
**Why:** Unvalidated input leads to type errors, injection, or data corruption.
**Detect:** Search for `createServerFn({ method: 'POST' })` without `.inputValidator()`.

```tsx
// ❌ BAD
createServerFn({ method: 'POST' }).handler(async ({ data }) => { ... })

// ✅ FIX
createServerFn({ method: 'POST' })
  .inputValidator(z.object({ title: z.string().min(1) }))
  .handler(async ({ data }) => { ... })
```

### AP-08: Editing routeTree.gen.ts

**What:** Manually modifying the auto-generated route tree file.
**Why:** The file is regenerated on every build. Manual changes are lost.
**Detect:** Check git diff on `routeTree.gen.ts` for manual edits.

### AP-09: Forgetting next() in Middleware

**What:** Middleware that doesn't call `next()`.
**Why:** Breaks the middleware chain. The request hangs or errors.
**Detect:** Search for `createMiddleware` handlers, verify each calls `return next()`.

```tsx
// ❌ BAD
createMiddleware().server(({ next }) => { console.log('logged') })

// ✅ FIX
createMiddleware().server(({ next }) => { console.log('logged'); return next() })
```

### AP-10: Throwing Instead of throw redirect

**What:** Returning redirect objects instead of throwing them.
**Why:** `redirect()` must be thrown, not returned. Returning it does nothing.
**Detect:** Search for `return redirect(` — should be `throw redirect(`.

```tsx
// ❌ BAD
return redirect({ to: '/login' })

// ✅ FIX
throw redirect({ to: '/login' })
```

---

## MEDIUM (Best Practices)

### AP-11: Missing Error/NotFound Components

**What:** Dynamic routes without `errorComponent` or `notFoundComponent`.
**Why:** Users see raw error messages or blank pages when data is missing.
**Detect:** Check routes with `$` params for missing error/notFound handlers.

### AP-12: Missing Head/Meta Tags

**What:** Routes without `head()` configuration.
**Why:** Poor SEO, missing page titles, no social sharing metadata.
**Detect:** Check page routes for missing `head()`.

### AP-13: No shellComponent on Root Route

**What:** Root route missing `shellComponent`.
**Why:** The HTML skeleton (`<html>`, `<head>`, `<body>`) may not render correctly. `<HeadContent />` and `<Scripts />` won't be placed properly.
**Detect:** Read `__root.tsx`, check for `shellComponent`.

### AP-14: Direct Database Access in Components

**What:** Importing database clients directly in route files or components.
**Why:** DB code can leak to client bundle. Should be behind `createServerFn`.
**Detect:** Search for `@prisma/client`, `drizzle`, `knex` imports in `routes/` files.

```tsx
// ❌ BAD (in route file)
import { db } from '~/lib/db'
loader: () => db.posts.findMany()

// ✅ FIX
import { getPosts } from '~/server/posts'  // uses createServerFn internally
loader: () => getPosts()
```

### AP-15: Not Using Import Protection

**What:** No `importProtection` in vite.config.ts.
**Why:** Server-only packages can accidentally end up in the client bundle.
**Detect:** Check vite.config.ts for `importProtection` config.

### AP-16: Inline Server Logic in Route Files

**What:** Complex server logic directly in route files instead of separate server function files.
**Why:** Hard to test, hard to reuse, bloats route files.
**Detect:** Long `createServerFn` definitions inside route files.

### AP-17: Missing Type Registration for Router

**What:** Not registering the router type globally.
**Why:** Loses type safety for `Link`, `useNavigate`, etc. across the app.

### AP-18: Client-Side Env Vars Without VITE_ Prefix

**What:** Trying to use `import.meta.env.MY_VAR` (without VITE_ prefix) in client code.
**Why:** Vite only exposes `VITE_`-prefixed variables to the client. Others are undefined.
**Detect:** Search for `import.meta.env.` followed by non-VITE_ names.

### AP-19: Missing Pending/Loading States

**What:** Routes with slow loaders but no `pendingComponent`.
**Why:** Users see nothing while data loads during navigation.
**Detect:** Routes with loaders but no `pendingComponent`.

### AP-20: Not Using scrollRestoration

**What:** Router created without `scrollRestoration: true`.
**Why:** Users lose scroll position when navigating back.
**Detect:** Check `createRouter` call in `router.tsx`.

---

## LOW (Code Quality)

### AP-21: Unnecessary SSR Disable

**What:** Using `ssr: false` on routes that could SSR fine.
**Why:** Hurts SEO and initial load performance unnecessarily.

### AP-22: Missing DevTools in Development

**What:** No `TanStackRouterDevtools` in root component.
**Why:** Harder to debug routing issues during development.

### AP-23: Hardcoded Paths Instead of Link

**What:** Using `<a href="/path">` instead of `<Link to="/path">`.
**Why:** Loses type safety, causes full page reload instead of client navigation.

### AP-24: No staleTime/gcTime on Data-Heavy Routes

**What:** Routes with heavy data fetching but no cache configuration.
**Why:** Unnecessary re-fetching on every navigation.

### AP-25: Using fetch() Instead of Server Functions

**What:** Using `fetch('/api/...')` in components instead of `useServerFn`.
**Why:** Loses type safety, no input validation, no middleware support.
