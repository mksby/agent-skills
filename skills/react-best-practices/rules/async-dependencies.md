---
title: Dependency-Based Parallelization
impact: CRITICAL
impactDescription: 2-10× improvement
tags: async, parallelization, dependencies
requires: better-all | none
---

## Dependency-Based Parallelization

For operations with partial dependencies, maximize parallelism by starting each task at the earliest possible moment.

**Problem:** Some async operations depend on others, but not all. Naive approaches either serialize everything or require complex manual orchestration.

**Incorrect (profile waits for config unnecessarily):**

```typescript
const [user, config] = await Promise.all([
  fetchUser(),
  fetchConfig()
])
const profile = await fetchProfile(user.id)  // config finished long ago
```

---

### Option 1: Using `better-all` (cleanest syntax)

```bash
npm install better-all
```

```typescript
import { all } from 'better-all'

const { user, config, profile } = await all({
  async user() { return fetchUser() },
  async config() { return fetchConfig() },
  async profile() {
    return fetchProfile((await this.$.user).id)
  }
})
```

`better-all` automatically detects dependencies and starts `config` and `profile` in parallel, where `profile` waits only for `user`.

Reference: [github.com/shuding/better-all](https://github.com/shuding/better-all)

---

### Option 2: Vanilla TypeScript (no dependencies)

Manual promise chaining with explicit dependency tracking:

```typescript
async function fetchAllData() {
  const userPromise = fetchUser()
  const configPromise = fetchConfig()

  // profile depends on user, but not on config
  const profilePromise = userPromise.then(user => fetchProfile(user.id))

  // await all at the end
  const [user, config, profile] = await Promise.all([
    userPromise,
    configPromise,
    profilePromise
  ])

  return { user, config, profile }
}
```

**For more complex dependencies:**

```typescript
async function fetchDashboardData() {
  // Level 1: no dependencies
  const userPromise = fetchUser()
  const configPromise = fetchConfig()
  const settingsPromise = fetchSettings()

  // Level 2: depends on user
  const profilePromise = userPromise.then(u => fetchProfile(u.id))
  const ordersPromise = userPromise.then(u => fetchOrders(u.id))

  // Level 3: depends on profile and config
  const recommendationsPromise = Promise.all([profilePromise, configPromise])
    .then(([profile, config]) => fetchRecommendations(profile.id, config.region))

  // Await all
  const [user, config, settings, profile, orders, recommendations] =
    await Promise.all([
      userPromise,
      configPromise,
      settingsPromise,
      profilePromise,
      ordersPromise,
      recommendationsPromise
    ])

  return { user, config, settings, profile, orders, recommendations }
}
```

---

### Visualization

```
Without optimization:
user -----> config -----> profile -----> done
            (waiting)     (waiting)
Total: 3 sequential calls

With dependency-based parallelization:
user -------> profile --->|
config ------------------>| done
Total: 2 sequential calls (user->profile runs parallel with config)
```

---

### When to use which

| Scenario | Recommendation |
|----------|----------------|
| 2-3 async calls | Manual Promise chaining |
| Complex dependency graph | `better-all` for readability |
| No new dependencies allowed | Manual approach |
