---
paths:
  - "**/*.ts"
  - "**/*.tsx"
---


- Enable strict mode in tsconfig.json — never disable it
- Never use any; prefer unknown for external data and narrow before use
- Prefer interfaces over type aliases for object shapes
- Use const assertions and readonly modifiers for immutable data
- Use satisfies operator for type-safe object literals
- Use Result/Option patterns instead of throwing for expected errors
- Use zod or valibot for runtime validation of external/user data
- Prefer named exports over default exports for better refactoring