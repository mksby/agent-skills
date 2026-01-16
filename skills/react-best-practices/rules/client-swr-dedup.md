---
title: Use Request Deduplication
impact: MEDIUM-HIGH
impactDescription: automatic deduplication
tags: client, deduplication, data-fetching
requires: swr | @tanstack/react-query | none
---

## Use Request Deduplication

Multiple component instances fetching the same data should share one request. This prevents redundant network calls and ensures data consistency.

**Incorrect (no deduplication, each instance fetches):**

```tsx
function UserList() {
  const [users, setUsers] = useState([])
  useEffect(() => {
    fetch('/api/users')
      .then(r => r.json())
      .then(setUsers)
  }, [])
}
```

---

### Option 1: Using `swr` (recommended, ~4kb)

```bash
npm install swr
```

```tsx
import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then(r => r.json())

function UserList() {
  const { data: users } = useSWR('/api/users', fetcher)
  return <ul>{users?.map(u => <li key={u.id}>{u.name}</li>)}</ul>
}
```

**For mutations:**

```tsx
import useSWRMutation from 'swr/mutation'

const updateUser = (url: string, { arg }: { arg: User }) =>
  fetch(url, { method: 'PUT', body: JSON.stringify(arg) })

function UpdateButton() {
  const { trigger } = useSWRMutation('/api/user', updateUser)
  return <button onClick={() => trigger({ name: 'New' })}>Update</button>
}
```

Reference: [swr.vercel.app](https://swr.vercel.app)

---

### Option 2: Using `@tanstack/react-query` (~12kb, more features)

```bash
npm install @tanstack/react-query
```

```tsx
import { useQuery, QueryClient, QueryClientProvider } from '@tanstack/react-query'

const queryClient = new QueryClient()

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <UserList />
    </QueryClientProvider>
  )
}

function UserList() {
  const { data: users } = useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(r => r.json())
  })
  return <ul>{users?.map(u => <li key={u.id}>{u.name}</li>)}</ul>
}
```

Reference: [tanstack.com/query](https://tanstack.com/query)

---

### Option 3: Vanilla React (no dependencies)

Custom hook with Map-based cache for simple deduplication:

```tsx
// lib/use-fetch.ts
const cache = new Map<string, { data: unknown; timestamp: number }>()
const inflight = new Map<string, Promise<unknown>>()
const CACHE_TTL = 30_000 // 30 seconds

export function useFetch<T>(url: string): {
  data: T | undefined
  isLoading: boolean
  error: Error | undefined
} {
  const [state, setState] = useState<{
    data: T | undefined
    isLoading: boolean
    error: Error | undefined
  }>({ data: undefined, isLoading: true, error: undefined })

  useEffect(() => {
    let cancelled = false

    async function fetchData() {
      const cached = cache.get(url)
      if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
        setState({ data: cached.data as T, isLoading: false, error: undefined })
        return
      }

      let promise = inflight.get(url)
      if (!promise) {
        promise = fetch(url).then(r => {
          if (!r.ok) throw new Error(r.statusText)
          return r.json()
        })
        inflight.set(url, promise)
      }

      try {
        const data = await promise
        cache.set(url, { data, timestamp: Date.now() })
        if (!cancelled) {
          setState({ data: data as T, isLoading: false, error: undefined })
        }
      } catch (error) {
        if (!cancelled) {
          setState({ data: undefined, isLoading: false, error: error as Error })
        }
      } finally {
        inflight.delete(url)
      }
    }

    fetchData()
    return () => { cancelled = true }
  }, [url])

  return state
}

// Usage
function UserList() {
  const { data: users, isLoading } = useFetch<User[]>('/api/users')
  if (isLoading) return <div>Loading...</div>
  return <ul>{users?.map(u => <li key={u.id}>{u.name}</li>)}</ul>
}
```

This provides basic deduplication without external dependencies. For production apps with complex caching needs, consider SWR or React Query.
