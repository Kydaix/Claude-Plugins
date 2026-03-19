# TanStack Start — Advanced Rendering Patterns Reference

## Error Boundaries in Detail

### Default Error Component on Router

```tsx
// src/router.tsx
import { createRouter, ErrorComponent } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

export function getRouter() {
  return createRouter({
    routeTree,
    defaultErrorComponent: ({ error, reset }) => (
      <ErrorComponent error={error} />
    ),
  })
}
```

### Per-Route Error Override

```tsx
import { createFileRoute, ErrorComponent } from '@tanstack/react-router'
import type { ErrorComponentProps } from '@tanstack/react-router'

function PostError({ error, reset }: ErrorComponentProps) {
  return (
    <div role="alert">
      <h2>Failed to load post</h2>
      <ErrorComponent error={error} />
      <button onClick={reset}>Retry</button>
    </div>
  )
}

export const Route = createFileRoute('/posts/$postId')({
  errorComponent: PostError,
  component: PostComponent,
})
```

### React Error Boundary Integration

```tsx
import { ErrorBoundary } from 'react-error-boundary'

function ErrorFallback({ error, resetErrorBoundary }: any) {
  console.error('[CLIENT ERROR]:', error)
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  )
}

export function App() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <Router />
    </ErrorBoundary>
  )
}
```

## TanStack Query Integration

### Root Route with QueryClient Context

```tsx
import { createRootRouteWithContext } from '@tanstack/react-router'
import type { QueryClient } from '@tanstack/react-query'

export const Route = createRootRouteWithContext<{ queryClient: QueryClient }>()({
  head: () => ({ /* meta tags */ }),
  component: RootComponent,
})
```

### Loader with ensureQueryData

```tsx
import { queryOptions } from '@tanstack/react-query'

const postQueryOptions = (postId: string) =>
  queryOptions({
    queryKey: ['post', postId],
    queryFn: () => fetchPost(postId),
  })

export const Route = createFileRoute('/posts/$postId')({
  loader: ({ context, params }) =>
    context.queryClient.ensureQueryData(postQueryOptions(params.postId)),
})

function Post() {
  const { postId } = Route.useParams()
  const { data } = useQuery(postQueryOptions(postId))
  // Automatically uses cached data from loader
}
```

### Benefits of TanStack Query + Start

- Loader pre-fetches data on the server
- `ensureQueryData` populates the query cache
- Component uses `useQuery` for reactivity and cache management
- Background refetching, optimistic updates, and mutations via React Query

## Multi-Tier Caching (ISR + Client)

Combine CDN caching with client-side caching:

```typescript
export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => fetchPost(params.postId),
  // CDN caching (via headers)
  headers: () => ({
    'Cache-Control': 'public, max-age=3600, stale-while-revalidate=86400',
  }),
  // Client-side caching (via TanStack Router)
  staleTime: 60_000,        // Fresh for 60s on client
  gcTime: 5 * 60_000,       // Keep in memory 5 minutes
})
```

## Form Handling Patterns

### FormData with Server Functions

```tsx
export const submitForm = createServerFn({ method: 'POST' })
  .inputValidator((data) => {
    if (!(data instanceof FormData)) throw new Error('Expected FormData')
    return {
      name: data.get('name')?.toString() || '',
      email: data.get('email')?.toString() || '',
    }
  })
  .handler(async ({ data }) => {
    return { success: true }
  })
```

### Form Component with Loading State

```tsx
function JokeForm() {
  const router = useRouter()
  const [question, setQuestion] = useState('')
  const [answer, setAnswer] = useState('')
  const [isSubmitting, setIsSubmitting] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    if (!question || !answer || isSubmitting) return
    try {
      setIsSubmitting(true)
      await addJoke({ data: { question, answer } })
      setQuestion('')
      setAnswer('')
      router.invalidate()  // Refresh route data
    } catch (err) {
      setError('Failed to add joke')
    } finally {
      setIsSubmitting(false)
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      {error && <div style={{ color: 'red' }}>{error}</div>}
      <input value={question} onChange={e => setQuestion(e.target.value)} required />
      <input value={answer} onChange={e => setAnswer(e.target.value)} required />
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Adding...' : 'Add'}
      </button>
    </form>
  )
}
```

### Key Pattern: router.invalidate()

After a mutation, call `router.invalidate()` to re-run loaders and refresh data:

```tsx
const router = useRouter()
await createPost({ data: formData })
router.invalidate()  // Re-fetches all active loaders
```

## Rendering Markdown / Blog Content

### Static Markdown with content-collections

```typescript
// content-collections.ts
import { defineCollection, defineConfig } from '@content-collections/core'
import matter from 'gray-matter'

const posts = defineCollection({
  name: 'posts',
  directory: './src/blog',
  include: '*.md',
  schema: (z) => ({
    title: z.string(),
    published: z.string().date(),
    description: z.string().optional(),
    authors: z.string().array(),
  }),
  transform: ({ content, ...post }) => {
    const { data, content: body } = matter(content)
    return { ...post, slug: post._meta.path, content: body }
  },
})

export default defineConfig({ collections: [posts] })
```

### Blog Post Route

```tsx
// src/routes/blog/$slug.tsx
import { createFileRoute, notFound } from '@tanstack/react-router'
import { allPosts } from 'content-collections'

export const Route = createFileRoute('/blog/$slug')({
  loader: ({ params }) => {
    const post = allPosts.find((p) => p.slug === params.slug)
    if (!post) throw notFound()
    return post
  },
  component: BlogPost,
})

function BlogPost() {
  const post = Route.useLoaderData()
  return (
    <article>
      <h1>{post.title}</h1>
      <p>By {post.authors.join(', ')} on {post.published}</p>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  )
}
```

Two approaches for markdown:
- **Static**: `content-collections` for build-time loading (blog posts in repo)
- **Dynamic**: Fetch at runtime from GitHub or any remote source
- Both use the `unified` ecosystem for processing markdown to HTML
