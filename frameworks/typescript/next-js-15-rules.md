---
paths:
  - "**/*.js"
  - "**/*.jsx"
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.svelte"
---

# Next.js 15 / React 19 + Shadcn/Radix Rules

Scope: conventions specific to the React 19 + Next.js 15 (App Router) + Shadcn UI + Radix UI + Tailwind CSS stack. For general Next.js practices see `next-js.md`; for general TypeScript/engineering process see `coding-style-rules.md` and `software-development/claude-coding-assistant-rules.md`.

## React 19
- Use the `use()` hook for reading promises/context in render instead of `useEffect` + state for simple async reads
- Use `useActionState` (replaces `useFormState`) for form actions with pending/error state
- Use the `useOptimistic` hook for instant UI feedback on server-action mutations
- Rely on the React Compiler's auto-memoization where enabled; don't hand-add `useMemo`/`useCallback` preemptively on a compiler-enabled project

## Shadcn UI
- Shadcn components are copied into the repo (`components/ui/`), not installed as a dependency — treat them as project code, safe to customize directly rather than wrapping
- Compose new components from existing `components/ui/` primitives rather than styling raw HTML elements when a suitable primitive exists
- Keep Shadcn's generated component API (props, variants via `cva`) intact when customizing so consumers aren't surprised

## Radix UI
- Radix primitives are unstyled/behavior-only — accessibility (focus trap, ARIA, keyboard nav) comes for free; don't re-implement it
- Use Radix's `asChild` prop to compose a primitive's behavior onto a custom element instead of wrapping in an extra DOM node
- Don't override Radix's internal ARIA attributes; if the default semantics are wrong for the use case, reach for a different primitive rather than fighting it

## Naming
- Use descriptive names with auxiliary verbs for boolean/state values (`isLoading`, `hasError`)
