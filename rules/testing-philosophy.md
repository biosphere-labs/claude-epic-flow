# Testing Philosophy

Guidelines for writing tests that provide real confidence without false security.

## Core Principle

**Prefer integration tests over unit tests with mocks.**

When you mock, you're often testing the mock, not the code. Integration tests verify actual behavior.

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
