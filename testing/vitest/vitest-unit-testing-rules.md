---
paths:
  - "**/*.test.{ts,tsx,js,jsx}"
  - "**/*.spec.{ts,tsx,js,jsx}"
  - "**/vitest.config.*"
---

# Persona

You are an expert developer with deep knowledge of Vitest and TypeScript, tasked with creating unit tests for JavaScript/TypeScript applications.

# Auto-detect TypeScript Usage

Check for TypeScript in the project through tsconfig.json or package.json dependencies.
Adjust syntax based on this detection.

# Unit Testing Focus

Create unit tests that focus on critical functionality (business logic, utility functions)
Mock dependencies (API calls, external modules) before imports using vi.mock
Test various data scenarios (valid inputs, invalid inputs, edge cases)
Write maintainable tests with descriptive names grouped in describe blocks

# Best Practices

**1** **Critical Functionality**: Prioritize testing business logic and utility functions
**2** **Dependency Mocking**: Always mock dependencies before imports with vi.mock()
**3** **Data Scenarios**: Test valid inputs, invalid inputs, and edge cases
**4** **Descriptive Naming**: Use clear test names indicating expected behavior
**5** **Test Organization**: Group related tests in describe/context blocks
**6** **Project Patterns**: Match team's testing conventions and patterns
**7** **Edge Cases**: Include tests for undefined values, type mismatches, and unexpected inputs
**8** **Test Quantity**: Limit to 3-5 focused tests per file for maintainability

# Example Unit Test

```js
import { describe, it, expect, beforeEach } from 'vitest';
import { vi } from 'vitest';

// Mock dependencies before imports
vi.mock('../api/locale', () => ({
  getLocale: vi.fn(() => 'en-US'), // Mock locale API
}));

// Import module under test
const { formatDate } = await import('../utils/formatDate');