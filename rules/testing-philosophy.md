# Testing Philosophy

Guidelines for writing tests that provide real confidence without false security.

## Core Principles

**Prefer integration tests over unit tests with mocks.**

When you mock, you're often testing the mock, not the code. Integration tests verify actual behavior.

## E2E Tests: Strongly Prefer Real Calls

**E2E tests should use real API calls, not mocks.** The whole point of E2E testing is verifying the complete system works together.

**Exception:** Use mocks when real calls have significant cost (>$1 per test run) - e.g., paid APIs, LLM calls, or external services that charge per request. In these cases, mock the expensive call but document why.

```typescript
// ❌ WRONG - This is pointless
jest.mock('../api/userService');
mockedUserService.getUser.mockResolvedValue({ name: 'Test' });
// Congratulations, you proved your mock works. The real API could be broken.

// ✅ CORRECT - Real calls to real backend
const response = await fetch('http://localhost:3000/api/users/1');
const user = await response.json();
expect(user.name).toBe('Test User');
```

### Why Mocks in E2E Are Usually Pointless

1. **You're testing the mock, not the code** - A mock always returns what you told it to. That proves nothing.
2. **Real bugs hide in real integrations** - Database queries, API serialization, auth flows - mocks skip all of this.
3. **False confidence** - Tests pass, you deploy, production breaks. The mock lied to you.

**When mocks ARE appropriate in E2E:**
- External paid APIs where each call costs money (LLM APIs, payment processors in non-sandbox mode)
- Rate-limited services that would block CI/CD
- Third-party services with no test/sandbox environment

When you do mock, add a comment explaining why the real call isn't feasible.

### What E2E Tests Should Do

E2E tests should:
- Start a real local backend (connected to staging/test database)
- Make real HTTP calls through the UI or API
- Verify real data flows through real systems
- Test actual user workflows end-to-end

```typescript
// Good E2E test - real browser, real backend, real database
test('user can create and view a project', async ({ page }) => {
  await page.goto('http://localhost:5173');
  await page.click('[data-testid="new-project"]');
  await page.fill('[name="title"]', 'My Project');
  await page.click('[type="submit"]');

  // Verify it was actually created in the real database
  await expect(page.locator('.project-title')).toHaveText('My Project');
});
```

### Value of Local E2E Tests

Even local E2E tests provide significant value:
- **If tests pass locally but fail on AWS**: You've narrowed down the bug to environment/deployment differences
- **If tests fail locally**: You catch the bug before it ever reaches CI/CD
- **Real integration coverage**: You know the code actually works with real services

## Test Hierarchy

### 1. Integration Tests (Preferred)
Test real interactions between components:
- API endpoints with real database
- Services calling real dependencies
- Components with real state management

```typescript
// Good: Real database interaction
it('creates user and retrieves it', async () => {
  const user = await userService.create({ name: 'Test' });
  const found = await userService.findById(user.id);
  expect(found.name).toBe('Test');
});
```

### 2. Pure Logic Unit Tests (When Appropriate)
Test functions with no external dependencies:
- Utility functions
- Transformations
- Calculations
- Validators

```typescript
// Good: Pure function, no dependencies
it('formats currency correctly', () => {
  expect(formatCurrency(1234.5)).toBe('$1,234.50');
});
```

### 3. E2E/Playwright Tests (For Regression)
Build up over time as features are completed:
- Critical user flows
- Features that broke before
- Complex multi-step interactions

**Remember: E2E tests NEVER use mocks. See "E2E Tests: NEVER Use Mocks" section above.**

### 4. Unit Tests with Mocks (Avoid)
Only when absolutely necessary:
- External APIs you don't control
- Paid services in CI
- Time-dependent code

```typescript
// Avoid: Testing the mock, not the code
jest.mock('../userService');
mockedUserService.findById.mockResolvedValue({ name: 'Test' });
// This test passes even if userService is broken
```

## Rules

### R-1: No Mocks for Internal Code
Never mock your own services, repositories, or utilities. If you need to mock it, you need an integration test.

### R-2: Separate Test Types
Keep pure-logic tests separate from integration tests:
```
tests/
  unit/        # Pure logic only
  integration/ # Database, API, services
  e2e/         # Playwright browser tests
```

### R-3: Build Regression Suite Over Time
When you fix a bug or complete a feature:
1. Write a test that would have caught it
2. Add to the permanent test suite
3. Run in CI going forward

### R-4: Test Real Behavior
Ask: "Does this test verify what users experience?"
- ✅ "User can log in and see dashboard"
- ❌ "LoginService.authenticate returns token when mock returns success"

### R-5: Prefer Fewer, Better Tests
One integration test that exercises a real flow beats ten unit tests with mocks.

## When to Write Each Type

| Scenario | Test Type |
|----------|-----------|
| New API endpoint | Integration test |
| Utility function | Unit test (pure logic) |
| Bug fix | Regression test (integration or E2E) |
| User flow | Playwright E2E |
| Algorithm | Unit test (pure logic) |
| Database operation | Integration test |
| Component with API calls | Integration test |

## Anti-Patterns

### ❌ Mocking Everything
```typescript
// Bad: Every dependency mocked
jest.mock('../db');
jest.mock('../cache');
jest.mock('../logger');
jest.mock('../config');
// What are we even testing?
```

### ❌ Testing Implementation Details
```typescript
// Bad: Breaks when refactored
expect(service.internalMethod).toHaveBeenCalledWith('x');
```

### ❌ Tests That Always Pass
```typescript
// Bad: Mock returns what you expect
mock.getData.mockResolvedValue(expectedData);
expect(await service.getData()).toEqual(expectedData);
// This test is worthless
```

### ❌ Mocking Internal APIs in E2E Tests
```typescript
// Bad: Mocking your own backend defeats E2E purpose
beforeEach(() => {
  jest.mock('../api/client');
  mockClient.fetchData.mockResolvedValue(testData);
});

test('displays data', async ({ page }) => {
  // This proves NOTHING - the real API could be completely broken
  await page.goto('/dashboard');
  await expect(page.locator('.data')).toBeVisible();
});
// The mock bypasses database, auth, serialization, network - all the things that break
```

### ✅ Acceptable: Mocking Expensive External APIs
```typescript
// OK: Mocking a paid LLM API to avoid $$ per test run
jest.mock('../services/openai', () => ({
  generateSummary: jest.fn().mockResolvedValue('Mock summary')
  // Real call costs $0.05 each, would add up in CI
}));
```

## Building the Regression Suite

### During Bug Fixes
1. Reproduce the bug manually
2. Write a test that fails (proves the bug)
3. Fix the bug
4. Test passes
5. Commit test with fix

### During Feature Work
1. Complete the feature
2. Write E2E test for the happy path
3. Write integration test for edge cases
4. Add to CI pipeline

### Over Time
The test suite grows organically:
- Each bug fix adds a regression test
- Each feature adds integration/E2E tests
- Pure logic gets unit tests
- Coverage increases without mock bloat

## Remember

> "A test that uses mocks is a test of your mocking skills, not your code."

The goal is confidence that the system works, not 100% coverage with tests that don't catch real bugs.
