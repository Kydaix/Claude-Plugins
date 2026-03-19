# Complete CRUD with Server Functions

A full CRUD implementation with Zod validation, server functions, and a React component.

## src/server/posts.ts

```typescript
import { createServerFn } from '@tanstack/react-start'
import { z } from 'zod'

// Schemas
const CreatePostSchema = z.object({
  title: z.string().min(1, 'Title is required').max(200),
  body: z.string().min(1, 'Body is required'),
})

const UpdatePostSchema = z.object({
  id: z.string(),
  title: z.string().min(1).max(200).optional(),
  body: z.string().min(1).optional(),
})

const IdSchema = z.object({ id: z.string() })

// READ - List all posts
export const getPosts = createServerFn({ method: 'GET' }).handler(async () => {
  return db.posts.findMany({ orderBy: { createdAt: 'desc' } })
})

// READ - Single post
export const getPost = createServerFn({ method: 'GET' })
  .inputValidator(IdSchema)
  .handler(async ({ data }) => {
    const post = await db.posts.findUnique({ where: { id: data.id } })
    if (!post) throw new Error('Post not found')
    return post
  })

// CREATE
export const createPost = createServerFn({ method: 'POST' })
  .inputValidator(CreatePostSchema)
  .handler(async ({ data }) => {
    return db.posts.create({
      data: {
        title: data.title,
        body: data.body,
      },
    })
  })

// UPDATE
export const updatePost = createServerFn({ method: 'POST' })
  .inputValidator(UpdatePostSchema)
  .handler(async ({ data }) => {
    const { id, ...updates } = data
    return db.posts.update({
      where: { id },
      data: updates,
    })
  })

// DELETE
export const deletePost = createServerFn({ method: 'POST' })
  .inputValidator(IdSchema)
  .handler(async ({ data }) => {
    await db.posts.delete({ where: { id: data.id } })
    return { success: true }
  })
```

## src/routes/posts/index.tsx — List + Create

```tsx
import { createFileRoute, Link } from '@tanstack/react-router'
import { useServerFn } from '@tanstack/react-start'
import { useState } from 'react'
import { getPosts, createPost, deletePost } from '~/server/posts'

export const Route = createFileRoute('/posts/')({
  loader: () => getPosts(),
  component: PostListPage,
})

function PostListPage() {
  const posts = Route.useLoaderData()
  const create = useServerFn(createPost)
  const remove = useServerFn(deletePost)
  const [title, setTitle] = useState('')
  const [body, setBody] = useState('')

  const handleCreate = async (e: React.FormEvent) => {
    e.preventDefault()
    await create({ data: { title, body } })
    setTitle('')
    setBody('')
    // Reload route data
    Route.router.invalidate()
  }

  const handleDelete = async (id: string) => {
    await remove({ data: { id } })
    Route.router.invalidate()
  }

  return (
    <div>
      <h1>Posts</h1>

      <form onSubmit={handleCreate}>
        <input
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          placeholder="Title"
          required
        />
        <textarea
          value={body}
          onChange={(e) => setBody(e.target.value)}
          placeholder="Body"
          required
        />
        <button type="submit">Create Post</button>
      </form>

      <ul>
        {posts.map((post) => (
          <li key={post.id}>
            <Link to="/posts/$postId" params={{ postId: post.id }}>
              {post.title}
            </Link>
            <button onClick={() => handleDelete(post.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

## src/routes/posts/$postId.tsx — Read + Update

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { useServerFn } from '@tanstack/react-start'
import { useState } from 'react'
import { getPost, updatePost } from '~/server/posts'

export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => getPost({ data: { id: params.postId } }),
  component: PostDetailPage,
  errorComponent: ({ error }) => <div>Error: {error.message}</div>,
  notFoundComponent: () => <div>Post not found</div>,
})

function PostDetailPage() {
  const post = Route.useLoaderData()
  const update = useServerFn(updatePost)
  const [editing, setEditing] = useState(false)
  const [title, setTitle] = useState(post.title)
  const [body, setBody] = useState(post.body)

  const handleUpdate = async (e: React.FormEvent) => {
    e.preventDefault()
    await update({ data: { id: post.id, title, body } })
    setEditing(false)
    Route.router.invalidate()
  }

  if (editing) {
    return (
      <form onSubmit={handleUpdate}>
        <input value={title} onChange={(e) => setTitle(e.target.value)} />
        <textarea value={body} onChange={(e) => setBody(e.target.value)} />
        <button type="submit">Save</button>
        <button type="button" onClick={() => setEditing(false)}>Cancel</button>
      </form>
    )
  }

  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
      <button onClick={() => setEditing(true)}>Edit</button>
    </div>
  )
}
```

## src/routes/api/posts.ts — Server Route (REST API)

```typescript
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/api/posts')({
  server: {
    handlers: {
      GET: async () => {
        const posts = await db.posts.findMany()
        return Response.json(posts)
      },
      POST: async ({ request }) => {
        const body = await request.json()
        const post = await db.posts.create({ data: body })
        return Response.json(post, { status: 201 })
      },
    },
  },
})
```
