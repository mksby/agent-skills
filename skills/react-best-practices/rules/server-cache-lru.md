---
title: Cross-Request LRU Caching
impact: HIGH
impactDescription: caches across requests
tags: server, cache, lru, cross-request
requires: lru-cache | none
---

## Cross-Request LRU Caching

`React.cache()` only works within one request. For data shared across sequential requests (user clicks button A then button B), use an LRU cache.

---

### Option 1: Using `lru-cache` (recommended)

```bash
npm install lru-cache
```

```typescript
import { LRUCache } from 'lru-cache'

const cache = new LRUCache<string, User>({
  max: 1000,
  ttl: 5 * 60 * 1000  // 5 minutes
})

export async function getUser(id: string) {
  const cached = cache.get(id)
  if (cached) return cached

  const user = await db.user.findUnique({ where: { id } })
  cache.set(id, user)
  return user
}
```

Reference: [github.com/isaacs/node-lru-cache](https://github.com/isaacs/node-lru-cache)

---

### Option 2: Vanilla TypeScript (no dependencies)

Simple Map-based cache with TTL and max size:

```typescript
// lib/cache.ts
interface CacheEntry<T> {
  value: T
  timestamp: number
}

export function createCache<T>(options: { max: number; ttl: number }) {
  const cache = new Map<string, CacheEntry<T>>()
  const { max, ttl } = options

  function evictExpired() {
    const now = Date.now()
    for (const [key, entry] of cache) {
      if (now - entry.timestamp > ttl) {
        cache.delete(key)
      }
    }
  }

  function evictOldest() {
    if (cache.size >= max) {
      const firstKey = cache.keys().next().value
      if (firstKey) cache.delete(firstKey)
    }
  }

  return {
    get(key: string): T | undefined {
      const entry = cache.get(key)
      if (!entry) return undefined
      if (Date.now() - entry.timestamp > ttl) {
        cache.delete(key)
        return undefined
      }
      return entry.value
    },

    set(key: string, value: T): void {
      evictExpired()
      evictOldest()
      cache.set(key, { value, timestamp: Date.now() })
    },

    delete(key: string): void {
      cache.delete(key)
    },

    clear(): void {
      cache.clear()
    }
  }
}

// Usage
const userCache = createCache<User>({ max: 1000, ttl: 5 * 60 * 1000 })

export async function getUser(id: string) {
  const cached = userCache.get(id)
  if (cached) return cached

  const user = await db.user.findUnique({ where: { id } })
  userCache.set(id, user)
  return user
}
```

---

### When to use which

| Scenario | Recommendation |
|----------|----------------|
| Simple caching, few entries | Vanilla Map solution |
| High-traffic, many entries | `lru-cache` (optimized eviction) |
| Distributed system | Redis or external cache |
| Vercel Fluid Compute | Either works, LRU cache recommended |

**With Vercel's [Fluid Compute](https://vercel.com/docs/fluid-compute):** LRU caching is effective because multiple concurrent requests share the same function instance. The cache persists without external storage.

**In traditional serverless:** Each invocation runs in isolation, so consider Redis for cross-process caching.
