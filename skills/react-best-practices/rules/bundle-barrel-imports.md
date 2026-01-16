---
title: Avoid Barrel File Imports
impact: CRITICAL
impactDescription: 200-800ms import cost, slow builds
tags: bundle, imports, tree-shaking, barrel-files, performance
requires: none
---

## Avoid Barrel File Imports

Import directly from source files instead of barrel files to avoid loading thousands of unused modules. **Barrel files** are entry points that re-export multiple modules (e.g., `index.js` that does `export * from './module'`).

Popular icon and component libraries can have **up to 10,000 re-exports** in their entry file. For many React packages, **it takes 200-800ms just to import them**, affecting both development speed and production cold starts.

**Why tree-shaking doesn't help:** When a library is marked as external (not bundled), the bundler can't optimize it. If you bundle it to enable tree-shaking, builds become substantially slower analyzing the entire module graph.

---

### Icon Libraries

**Incorrect (imports entire library):**

```tsx
import { Check, X, Menu } from 'lucide-react'
// Loads 1,583 modules, takes ~2.8s extra in dev
```

**Correct — if using `lucide-react`:**

```tsx
import Check from 'lucide-react/dist/esm/icons/check'
import X from 'lucide-react/dist/esm/icons/x'
import Menu from 'lucide-react/dist/esm/icons/menu'
// Loads only 3 modules (~2KB vs ~1MB)
```

**Alternative — `@heroicons/react` (better structure):**

```bash
npm install @heroicons/react
```

```tsx
// Already uses direct exports, no barrel file problem
import { CheckIcon, XMarkIcon, Bars3Icon } from '@heroicons/react/24/outline'
```

**Alternative — inline SVG (no dependencies):**

```tsx
function CheckIcon({ className }: { className?: string }) {
  return (
    <svg className={className} viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth={2}>
      <path strokeLinecap="round" strokeLinejoin="round" d="M5 13l4 4L19 7" />
    </svg>
  )
}
```

---

### Component Libraries

**Incorrect (imports entire library):**

```tsx
import { Button, TextField } from '@mui/material'
// Loads 2,225 modules, takes ~4.2s extra in dev
```

**Correct — if using `@mui/material`:**

```tsx
import Button from '@mui/material/Button'
import TextField from '@mui/material/TextField'
// Loads only what you use
```

**Alternative — `@radix-ui` (modular by design):**

```bash
npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu
```

```tsx
// Each package is separate, no barrel file problem
import * as Dialog from '@radix-ui/react-dialog'
import * as DropdownMenu from '@radix-ui/react-dropdown-menu'
```

**Alternative — native HTML + CSS (no dependencies):**

```tsx
function Button({ children, variant = 'primary', ...props }: ButtonProps) {
  return (
    <button className={`btn btn-${variant}`} {...props}>
      {children}
    </button>
  )
}
```

---

### Next.js 13.5+ Solution

If you must use barrel imports for ergonomics:

```js
// next.config.js
module.exports = {
  experimental: {
    optimizePackageImports: ['lucide-react', '@mui/material']
  }
}

// Then barrel imports work efficiently:
import { Check, X, Menu } from 'lucide-react'
// Automatically transformed to direct imports at build time
```

---

### When to use which

| Scenario | Recommendation |
|----------|----------------|
| Next.js 13.5+ | Use `optimizePackageImports` config |
| Other frameworks | Use direct imports |
| Few icons needed | Inline SVG (zero deps) |
| Many icons | `@heroicons/react` or direct lucide imports |
| Need UI components | `@radix-ui` (modular) or direct MUI imports |
| Simple UI | Native HTML elements + CSS |

---

### Libraries commonly affected

`lucide-react`, `@mui/material`, `@mui/icons-material`, `@tabler/icons-react`, `react-icons`, `@headlessui/react`, `@radix-ui/react-*`, `lodash`, `ramda`, `date-fns`, `rxjs`, `react-use`

Reference: [How we optimized package imports in Next.js](https://vercel.com/blog/how-we-optimized-package-imports-in-next-js)
