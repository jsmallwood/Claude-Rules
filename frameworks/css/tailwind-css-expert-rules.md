---
paths:
  - "**/*.css"
  - "**/*.tsx"
  - "**/*.jsx"
  - "**/*.html"
  - "**/*.svelte"
  - "**/tailwind.config.*"
---

You are Claude, an expert in Tailwind CSS, responsive design, and component styling.

## Class Organization
Organize classes in this order:
1. Layout (display, position, flex, grid)
2. Sizing (width, height, max-width)
3. Spacing (padding, margin, gap)
4. Typography (font, text)
5. Colors (background, text color, border color)
6. Borders (border, rounded)
7. Effects (shadow, opacity)
8. Transitions

## Responsive Design
```html
<!-- Mobile-first approach -->
<div class="
  flex flex-col gap-4 p-4
  sm:flex-row sm:gap-6
  lg:gap-8 lg:p-8
">
  <aside class="
    w-full
    sm:w-64
    lg:w-80
  ">
    Sidebar
  </aside>
  <main class="flex-1">
    Content
  </main>
</div>
```

## Dark Mode
```html
<div class="
  bg-white dark:bg-gray-900
  text-gray-900 dark:text-white
  border border-gray-200 dark:border-gray-700
">
  <p class="text-gray-600 dark:text-gray-400">
    Muted text
  </p>
</div>
```

## Component Variants
```tsx
import { cva, type VariantProps } from 'class-variance-authority';

const button = cva(
  'inline-flex items-center justify-center rounded-md font-medium transition-colors focus:outline-none focus:ring-2 disabled:opacity-50',
  {
    variants: {
      variant: {
        primary: 'bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500',
        secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200 focus:ring-gray-500',
        ghost: 'hover:bg-gray-100 focus:ring-gray-500',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4',
        lg: 'h-12 px-6 text-lg',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);
```


## Custom Configuration
```javascript
// tailwind.config.js
module.exports = {
  content: ['./src/**/*.{js,ts,jsx,tsx}'],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#f0f9ff',
          500: '#0ea5e9',
          900: '#0c4a6e',
        },
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
        mono: ['JetBrains Mono', 'monospace'],
      },
      animation: {
        'fade-in': 'fadeIn 0.5s ease-out',
        'slide-up': 'slideUp 0.3s ease-out',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideUp: {
          '0%': { transform: 'translateY(10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
};
```