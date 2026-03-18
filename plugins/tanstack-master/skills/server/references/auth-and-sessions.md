# TanStack Start — Authentication & Sessions Reference

## Session Management

TanStack Start provides `useSession` for managing HTTP-only cookie sessions.

### Session Setup

```typescript
// utils/session.ts
import { useSession } from '@tanstack/react-start/server'

type SessionData = {
  userId?: string
  email?: string
  role?: string
}

export function useAppSession() {
  return useSession<SessionData>({
    name: 'app-session',
    password: process.env.SESSION_SECRET!,  // At least 32 characters
    cookie: {
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      httpOnly: true,
    },
  })
}
```

### Session API

```typescript
const session = await useAppSession()

// Read session data
const userId = session.data.userId

// Update session
await session.update({
  userId: user.id,
  email: user.email,
  role: user.role,
})

// Clear session (logout)
await session.clear()
```

## Authentication Server Functions

### Login

```tsx
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'

export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string; password: string }) => data)
  .handler(async ({ data }) => {
    const user = await authenticateUser(data.email, data.password)

    if (!user) {
      return { error: 'Invalid credentials' }
    }

    const session = await useAppSession()
    await session.update({
      userId: user.id,
      email: user.email,
    })

    throw redirect({ to: '/dashboard' })
  })
```

### Logout

```tsx
export const logoutFn = createServerFn({ method: 'POST' }).handler(async () => {
  const session = await useAppSession()
  await session.clear()
  throw redirect({ to: '/' })
})
```

### Get Current User

```tsx
export const getCurrentUserFn = createServerFn({ method: 'GET' }).handler(
  async () => {
    const session = await useAppSession()
    const userId = session.data.userId

    if (!userId) return null

    return await getUserById(userId)
  },
)
```

## Route Protection

### Using beforeLoad

```tsx
export const Route = createFileRoute('/dashboard')({
  beforeLoad: async () => {
    const user = await getCurrentUserFn()
    if (!user) {
      throw redirect({ to: '/login' })
    }
    return { user }
  },
  component: Dashboard,
})

function Dashboard() {
  const { user } = Route.useRouteContext()
  return <h1>Welcome, {user.name}</h1>
}
```

### Using Auth Middleware

```tsx
import { createMiddleware } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'

export const authMiddleware = createMiddleware({ type: 'function' })
  .client(async ({ next }) => {
    const token = localStorage.getItem('auth-token')
    if (token) {
      return next({
        headers: { Authorization: `Bearer ${token}` },
      })
    }
    return next()
  })
  .server(async ({ next }) => {
    const session = await useAppSession()
    if (!session.data.userId) {
      throw redirect({ to: '/login' })
    }
    const user = await getUserById(session.data.userId)
    return next({ context: { user } })
  })

// Use in server functions
export const getProtectedData = createServerFn()
  .middleware([authMiddleware])
  .handler(async ({ context }) => {
    // context.user is guaranteed to exist
    return db.data.findMany({ where: { userId: context.user.id } })
  })
```

### Admin-Only Routes

```tsx
export const adminMiddleware = createMiddleware({ type: 'function' })
  .middleware([authMiddleware])
  .server(async ({ next, context }) => {
    if (context.user.role !== 'admin') {
      throw redirect({ to: '/unauthorized' })
    }
    return next()
  })

export const getAdminStats = createServerFn()
  .middleware([adminMiddleware])
  .handler(async ({ context }) => {
    return db.stats.getAll()
  })
```

## Login Form Component

```tsx
function LoginForm() {
  const [error, setError] = useState<string | null>(null)
  const login = useServerFn(loginFn)

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    const formData = new FormData(e.currentTarget)
    const result = await login({
      data: {
        email: formData.get('email') as string,
        password: formData.get('password') as string,
      },
    })
    if (result?.error) {
      setError(result.error)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" required />
      <input name="password" type="password" required />
      {error && <p>{error}</p>}
      <button type="submit">Login</button>
    </form>
  )
}
```

## Migration from Other Frameworks

### From Next.js

- Replace API routes with server functions
- Migrate NextAuth sessions to `useSession`
- Use `beforeLoad` instead of `getServerSideProps` for auth checks

### From Remix

- Convert loaders/actions to `createServerFn`
- Adapt session patterns to `useSession`
- Use middleware instead of Remix's `loader` for auth guards

## Best Practices

1. **Session secret must be at least 32 characters** — use a strong random string
2. **Set `httpOnly: true` on session cookies** — prevents XSS access
3. **Set `secure: true` in production** — cookies only sent over HTTPS
4. **Use `sameSite: 'lax'`** — prevents CSRF while allowing normal navigation
5. **Check auth in `beforeLoad`** — blocks the route before loader runs
6. **Use middleware for repeated auth patterns** — compose auth → admin → handler
7. **Throw `redirect()`, don't return it** — ensures immediate redirect
8. **Clear sessions on logout** — don't just delete the cookie, clear the session data
