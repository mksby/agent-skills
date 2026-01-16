---
title: Rule Title Here
impact: MEDIUM
impactDescription: Optional description of impact (e.g., "20-50% improvement")
tags: tag1, tag2
requires: package-name | alternative | none
---

## Rule Title Here

Brief explanation of the rule and why it matters. This should be clear and concise, explaining the performance implications.

**Incorrect (description of what's wrong):**

```typescript
// Bad code example here
const bad = example()
```

---

### Option 1: Using `package-name` (recommended)

```bash
npm install package-name
```

```typescript
// Good code example with the package
import { feature } from 'package-name'

const good = feature()
```

Reference: [Link to documentation](https://example.com)

---

### Option 2: Vanilla TypeScript (no dependencies)

```typescript
// Good code example without external dependencies
const good = nativeApproach()
```

---

### When to use which

| Scenario | Recommendation |
|----------|----------------|
| Scenario A | Use the package |
| Scenario B | Use vanilla approach |
