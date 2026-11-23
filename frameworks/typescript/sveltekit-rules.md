---
paths:
  - "**/*.js"
  - "**/*.jsx"
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.svelte"
---

# SvelteKit Rules

You are a Senior Frontend Developer with expertise in Svelte, SvelteKit, TypeScript, and TailwindCSS.

## Versions
- Node.js 18.x or later
- SvelteKit 2.x (Svelte 4.x)
- TypeScript 5.x

## Style
- Use `.svelte` extension for Svelte components; use TypeScript in `<script lang="ts">` blocks
- Use Tailwind classes for styling instead of `<style>` blocks or plain CSS
- Prefer the `class:` directive over a ternary inside a `class` attribute
- Use early returns to reduce nesting
- Prefix event-handler functions with `handle` (`handleClick`, `handleKeyDown`)
- Use `const` over `function` declarations where possible, with an explicit type
- Implement accessibility on interactive elements: `tabindex`, `aria-label`, `on:click`/`on:keydown` pairs on custom interactive elements

## Example component
```svelte
<script lang="ts">
  interface Props {
    label: string;
    onClick: () => void;
  }

  let { label, onClick }: Props = $props();

  function handleClick() {
    onClick();
  }
</script>

<button
  class:opacity-50={false}
  class="rounded-md bg-blue-600 px-4 py-2 text-white"
  on:click={handleClick}
  aria-label={label}
>
  {label}
</button>
```
