# Complete Authentication Flow

A full auth implementation: session setup, login/logout server functions, middleware, protected routes, and login form.

## src/utils/session.ts

```typescript
import { useSession } from '@tanstack/react-start/server'

type SessionData = {
  userId?: string
  email?: string
  role?: 'user' | 'admin'
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

## src/server/auth.ts

```typescript
import { createServerFn } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'
import { useAppSession } from '~/utils/session'

// Login
export const loginFn = createServerFn({ method: 'POST' })
  .inputValidator((data: { email: string; password: string }) => data)
  .handler(async ({ data }) => {
    // Replace with your auth logic (bcrypt, db lookup, etc.)
    if (data.email !== 'admin@example.com' || data.password !== 'password') {
      return { error: 'Invalid credentials' }
    }

    const session = await useAppSession()
    await session.update({
      userId: '1',
      email: data.email,
      role: 'admin',
    })

    throw redirect({ to: '/dashboard' })
  })

// Logout
export const logoutFn = createServerFn({ method: 'POST' }).handler(async () => {
  const session = await useAppSession()
  await session.clear()
  throw redirect({ to: '/' })
})

// Get current user
export const getCurrentUser = createServerFn({ method: 'GET' }).handler(
  async () => {
    const session = await useAppSession()
    if (!session.data.userId) return null
    return {
      id: session.data.userId,
      email: session.data.email!,
      role: session.data.role!,
    }
  },
)
```

## src/middleware/auth.ts

```typescript
import { createMiddleware } from '@tanstack/react-start'
import { redirect } from '@tanstack/react-router'
import { useAppSession } from '~/utils/session'

export const authMiddleware = createMiddleware({ type: 'function' })
  .server(async ({ next }) => {
    const session = await useAppSession()
    if (!session.data.userId) {
      throw redirect({ to: '/login' })
    }
    return next({
      context: {
        user: {
          id: session.data.userId,
          email: session.data.email!,
          role: session.data.role!,
        },
      },
    })
  })

export const adminMiddleware = createMiddleware({ type: 'function' })
  .middleware([authMiddleware])
  .server(async ({ next, context }) => {
    if (context.user.role !== 'admin') {
      throw redirect({ to: '/unauthorized' })
    }
    return next()
  })
```

## src/routes/login.tsx

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { useServerFn } from '@tanstack/react-start'
import { useState } from 'react'
import { loginFn } from '~/server/auth'

export const Route = createFileRoute('/login')({
  component: LoginPage,
})

function LoginPage() {
  const [error, setError] = useState<string | null>(null)
  const login = useServerFn(loginFn)

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    setError(null)
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
    <div>
      <h1>Login</h1>
      <form onSubmit={handleSubmit}>
        <div>
          <label htmlFor="email">Email</label>
          <input id="email" name="email" type="email" required />
        </div>
        <div>
          <label htmlFor="password">Password</label>
          <input id="password" name="password" type="password" required />
        </div>
        {error && <p style={{ color: 'red' }}>{error}</p>}
        <button type="submit">Login</button>
      </form>
    </div>
  )
}
```

## src/routes/dashboard.tsx

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { getCurrentUser, logoutFn } from '~/server/auth'
import { useServerFn } from '@tanstack/react-start'

export const Route = createFileRoute('/dashboard')({
  beforeLoad: async () => {
    const user = await getCurrentUser()
    if (!user) {
      throw redirect({ to: '/login' })
    }
    return { user }
  },
  component: DashboardPage,
})

function DashboardPage() {
  const { user } = Route.useRouteContext()
  const logout = useServerFn(logoutFn)

  return (
    <div>
      <h1>Dashboard</h1>
      <p>Welcome, {user.email} (role: {user.role})</p>
      <button onClick={() => logout()}>Logout</button>
    </div>
  )
}
```

## src/server/protected-data.ts (using middleware)

```typescript
import { createServerFn } from '@tanstack/react-start'
import { authMiddleware, adminMiddleware } from '~/middleware/auth'

// Any authenticated user
export const getUserData = createServerFn({ method: 'GET' })
  .middleware([authMiddleware])
  .handler(async ({ context }) => {
    // context.user is guaranteed by authMiddleware
    return { message: `Hello ${context.user.email}` }
  })

// Admin only
export const getAdminStats = createServerFn({ method: 'GET' })
  .middleware([adminMiddleware])
  .handler(async ({ context }) => {
    return { totalUsers: 42, revenue: 12345 }
  })
```
