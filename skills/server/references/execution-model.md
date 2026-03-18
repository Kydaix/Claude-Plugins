# TanStack Start — Execution Model Reference

## Where Code Runs

Understanding where code executes is critical in TanStack Start. Different parts of the application run in different environments.

## Execution Boundary Functions

### createServerFn — RPC (Server, callable from client)

The primary server function. Creates an RPC endpoint that runs on the server but can be called from both server and client code.

```tsx
import { createServerFn } from '@tanstack/react-start'

const updateUser = createServerFn({ method: 'POST' })
  .inputValidator((data: UserData) => data)
  .handler(async ({ data }) => {
    // Only runs on server, but client can call it via RPC
    return await db.users.update(data)
  })
```

### createServerOnlyFn — Server utility (crashes on client)

For utility functions that must NEVER run on the client. If called from client code, it crashes immediately.

```tsx
import { createServerOnlyFn } from '@tanstack/react-start'

const getEnvVar = createServerOnlyFn(() => process.env.DATABASE_URL)
```

Use cases: accessing env vars, file system operations, internal utilities.

### createClientOnlyFn — Client utility (crashes on server)

For code that must NEVER run on the server. If called from server code, it crashes.

```tsx
import { createClientOnlyFn } from '@tanstack/react-start'

const saveToStorage = createClientOnlyFn((data: any) => {
  localStorage.setItem('data', JSON.stringify(data))
})
```

Use cases: localStorage, sessionStorage, DOM manipulation, browser APIs.

### createIsomorphicFn — Different implementations per environment

Provides separate implementations for server and client:

```tsx
import { createIsomorphicFn } from '@tanstack/react-start'

const logger = createIsomorphicFn()
  .server((msg: string) => console.log(`[SERVER]: ${msg}`))
  .client((msg: string) => console.log(`[CLIENT]: ${msg}`))

// Works on both server and client with different behavior
logger('Hello')
```

Use cases: logging, analytics, error reporting.

## Code Execution by Location

### Components — Client (and server during SSR)

Components render on the server during SSR, then hydrate on the client. After hydration, they run exclusively on the client.

```tsx
function MyComponent() {
  // Runs on server (SSR) then client (hydration + interaction)
  const [count, setCount] = useState(0)
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

### Loaders — Server (SSR) AND Client (navigation)

**CRITICAL: Loaders run on BOTH server and client.** During SSR, the loader runs on the server. During client-side navigation, it runs on the client.

```tsx
// ❌ DANGEROUS — runs on client too!
export const Route = createFileRoute('/users')({
  loader: () => {
    const secret = process.env.SECRET  // EXPOSED on client!
    return fetch(`/api?key=${secret}`)
  },
})

// ✅ SAFE — server function for server-only logic
const getUsers = createServerFn().handler(() => {
  const secret = process.env.SECRET  // Server only
  return fetch(`/api?key=${secret}`)
})

export const Route = createFileRoute('/users')({
  loader: () => getUsers(),
})
```

### beforeLoad — Server (SSR) AND Client (navigation)

Same as loaders — runs on both environments:

```tsx
export const Route = createFileRoute('/dashboard')({
  beforeLoad: async () => {
    // Use server function for auth checks
    const user = await requireAuth()
    return { user }
  },
})
```

### Server Functions — Server only

`createServerFn` handlers always run on the server:

```tsx
const getData = createServerFn().handler(async () => {
  // Always server — safe to use process.env, db, etc.
  return db.query('SELECT * FROM users')
})
```

### Server Routes — Server only

HTTP handlers in `server.handlers` always run on the server:

```tsx
export const Route = createFileRoute('/api/data')({
  server: {
    handlers: {
      GET: async () => {
        // Always server
        return Response.json(await db.getData())
      },
    },
  },
})
```

## Summary Table

| Location | Server (SSR) | Client (Hydration) | Client (Navigation) |
|----------|:---:|:---:|:---:|
| Components | ✅ | ✅ | ✅ |
| Loaders | ✅ | ❌ | ✅ |
| beforeLoad | ✅ | ❌ | ✅ |
| Server Functions | ✅ | ✅ (via RPC) | ✅ (via RPC) |
| Server Routes | ✅ | ❌ | ❌ |
| head() | ✅ | ❌ | ✅ |

## Common Pitfalls

### Pitfall 1: Using process.env in loaders

```tsx
// ❌ Secret leaks to client
loader: () => db.connect(process.env.DATABASE_URL)

// ✅ Wrap in server function
const getData = createServerFn().handler(() => db.connect(process.env.DATABASE_URL))
loader: () => getData()
```

### Pitfall 2: Using browser APIs during SSR

```tsx
// ❌ Crashes on server
function MyComponent() {
  const width = window.innerWidth  // window undefined on server
}

// ✅ Use ClientOnly or check environment
import { ClientOnly } from '@tanstack/react-router'
<ClientOnly fallback={<span>Loading...</span>}>
  <BrowserOnlyComponent />
</ClientOnly>
```

### Pitfall 3: Assuming loaders are server-only

This is the most common mistake. **Always use `createServerFn` for any server-only logic in loaders.**
