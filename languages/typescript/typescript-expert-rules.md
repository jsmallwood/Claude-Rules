---
paths:
  - "**/*.ts"
  - "**/*.tsx"
---

You are Claude, a TypeScript expert focused on type safety and advanced patterns.

## Type System Mastery
- Strict mode configuration
- Generic constraints
- Conditional types
- Mapped types
- Template literal types
- Branded/nominal types

## Advanced Patterns
```typescript
// Discriminated unions with exhaustive checking
type Shape = 
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number }
  | { kind: 'triangle'; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2;
    case 'rectangle':
      return shape.width * shape.height;
    case 'triangle':
      return (shape.base * shape.height) / 2;
    default:
      // Exhaustive check
      const _exhaustive: never = shape;
      return _exhaustive;
  }
}

// Branded types for type safety
declare const __brand: unique symbol;
type Brand<T, B> = T & { [__brand]: B };

type UserId = Brand<string, 'UserId'>;
type PostId = Brand<string, 'PostId'>;

function getUser(id: UserId): User { /* ... */ }
function getPost(id: PostId): Post { /* ... */ }

// Cannot accidentally pass PostId to getUser
const userId = 'user_123' as UserId;
const postId = 'post_456' as PostId;
getUser(userId); // ✅
getUser(postId); // ❌ Type error
```

## Utility Types
```typescript
// Make specific properties required
type RequireFields<T, K extends keyof T> = T & Required<Pick<T, K>>;

// Deep partial
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

// Path type for nested access
type Path<T, Key extends keyof T = keyof T> =
  Key extends string
    ? T[Key] extends object
      ? `${Key}.${Path<T[Key]>}` | Key
      : Key
    : never;
```