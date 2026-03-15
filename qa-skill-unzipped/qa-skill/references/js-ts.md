# JavaScript / TypeScript QA Reference

## Detecting the Test Framework

Check `package.json` devDependencies and scripts:

| Dependency | Framework | Run command |
|-----------|-----------|-------------|
| `jest` | Jest | `npx jest` or `npm test` |
| `vitest` | Vitest | `npx vitest run` |
| `mocha` | Mocha | `npx mocha` |
| `@playwright/test` | Playwright (E2E) | `npx playwright test` |
| `cypress` | Cypress (E2E) | `npx cypress run` |

Always use the command defined in `package.json` scripts — not raw `npx` — to respect project config.

## When No Test Script Exists

If `package.json` has no `test` script, do NOT invent one or fail the run. Instead:

1. Note the gap in the QA Summary Report: `⚠️ no test script — unit tests not yet configured`
2. Run `<pm> run build` + `<pm> run typecheck` as the minimum quality floor
3. If asked to fix it: recommend adding Vitest (preferred for Next.js/Vite projects) or Jest, but **ask the user first** — do not add deps unilaterally

```bash
# Minimum quality gate when no test script exists
<pm> run typecheck   # catches type errors
<pm> run lint        # catches style/logic issues
<pm> run build       # catches build-time errors (env vars, RSC constraints, etc.)
```

## Build Verification

Always run the production build as part of QA — `tsc --noEmit` passes but `next build` can still fail (missing env vars, dynamic import issues, RSC constraints, etc.):

```bash
# Next.js
npm run build        # runs next build

# Generic
npx tsc --noEmit     # type check only (no build artifacts)
```

## Running Tests

```bash
# Run all tests (use actual script name from package.json)
npm test
pnpm test
yarn test

# Watch mode (dev only, not for CI/QA)
npm run test:watch

# Run a specific file
npx vitest run src/utils/parser.test.ts
npx jest src/utils/parser.test.ts

# Run tests matching a pattern
npx jest --testNamePattern="should parse"
npx vitest run --reporter=verbose -t "should parse"

# With coverage
npx vitest run --coverage
npx jest --coverage
```

## Typecheck

```bash
# CI-safe typecheck (no emit)
npx tsc --noEmit

# With a specific tsconfig
npx tsc --noEmit -p tsconfig.test.json
```

## Linting

```bash
# ESLint
npx eslint src/
npx eslint src/ --ext .ts,.tsx

# Auto-fix
npx eslint src/ --fix

# Prettier check
npx prettier --check src/

# Biome (newer alternative)
npx biome check src/
```

## Writing Unit Tests

### Vitest / Jest (co-located style)

```typescript
// src/utils/math.test.ts
import { describe, it, expect, vi } from 'vitest' // or from '@jest/globals'
import { add, divide } from './math'

describe('add', () => {
  it('returns sum of two numbers', () => {
    expect(add(1, 2)).toBe(3)
  })

  it('handles negative numbers', () => {
    expect(add(-1, -2)).toBe(-3)
  })
})

describe('divide', () => {
  it('throws on division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero')
  })
})
```

### Async tests

```typescript
it('fetches user data', async () => {
  const user = await fetchUser(1)
  expect(user.id).toBe(1)
  expect(user.name).toBeDefined()
})
```

### Mocking

```typescript
// Vitest
import { vi } from 'vitest'

vi.mock('../api/client', () => ({
  fetchUser: vi.fn().mockResolvedValue({ id: 1, name: 'Alice' })
}))

// Jest
jest.mock('../api/client', () => ({
  fetchUser: jest.fn().mockResolvedValue({ id: 1, name: 'Alice' })
}))
```

### React component tests (Testing Library)

```typescript
import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from './Button'

it('calls onClick when clicked', () => {
  const onClick = vi.fn()
  render(<Button onClick={onClick}>Click me</Button>)
  fireEvent.click(screen.getByText('Click me'))
  expect(onClick).toHaveBeenCalledOnce()
})
```

## E2E Tests — Playwright

```bash
# Run all E2E tests
npx playwright test

# Run headed (visible browser)
npx playwright test --headed

# Run specific file
npx playwright test e2e/login.spec.ts

# Debug mode
npx playwright test --debug

# Show report
npx playwright show-report
```

### Writing Playwright tests

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test'

test('user can log in', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[name=email]', 'user@example.com')
  await page.fill('[name=password]', 'password')
  await page.click('button[type=submit]')
  await expect(page).toHaveURL('/dashboard')
  await expect(page.locator('h1')).toContainText('Welcome')
})
```

## Common Failure Patterns

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `Cannot find module` | Wrong import path or missing dep | Check path, run `npm install` |
| `Type X is not assignable to Y` | TS type mismatch | Fix types or add assertion |
| `act()` warning in React tests | State update outside act | Wrap in `act()` or use `userEvent` |
| Flaky async test | Missing `await` or wrong timeout | Add `await`, increase `timeout` |
| Playwright timeout | Page not loaded / selector wrong | Use `waitForSelector`, check locator |
| ESLint `no-explicit-any` | Untyped variable | Add proper type |
| Mock not reset between tests | Shared mock state | Add `beforeEach(() => vi.clearAllMocks())` |

## Config Files to Check

| File | Purpose |
|------|---------|
| `vitest.config.ts` | Vitest setup, coverage, globals |
| `jest.config.ts` | Jest setup, transform, moduleNameMapper |
| `tsconfig.json` | TypeScript compiler options |
| `.eslintrc` / `eslint.config.js` | Lint rules |
| `playwright.config.ts` | E2E base URL, browser, retries |
