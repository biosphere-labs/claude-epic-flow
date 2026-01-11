---
allowed-tools: Bash, Read, Write, Glob, Grep, mcp__playwright__browser_navigate, mcp__playwright__browser_snapshot, mcp__playwright__browser_click, mcp__playwright__browser_type, mcp__playwright__browser_take_screenshot
---

# Acceptance Test Generator

Generate Playwright tests from acceptance criteria, building up regression suite over time.

## CRITICAL: No Mocks

**E2E tests must NEVER use mocks.** See `/rules/testing-philosophy.md`.

Generated tests must:
- Call real backend APIs (running locally against staging/test database)
- Use real authentication flows
- Interact with real services
- NEVER use jest.mock, mockResolvedValue, or any mocking pattern

Mocks in E2E tests prove nothing - they only verify the mock works, not the actual code.

## Usage
```
/testing:acceptance <epic-or-feature>
```

**Examples:**
```bash
/testing:acceptance auth-flow          # From epic acceptance criteria
/testing:acceptance "user can login"   # From description
```

## Instructions

### 1. Find Acceptance Criteria

Check for acceptance criteria in order:
```bash
# From epic
cat workflow/epics/$ARGUMENTS/epic.md | grep -A 50 "## Acceptance Criteria"

# From PRD
cat workflow/prds/$ARGUMENTS.md | grep -A 50 "## Success Criteria"
```

If no file found, use $ARGUMENTS as the acceptance criteria description.

### 2. Parse Criteria into Test Cases

For each acceptance criterion, create a test case:

```
AC: User can log in with valid credentials
→ Test: "should allow login with valid email and password"

AC: Error shown for invalid password
→ Test: "should show error message for incorrect password"

AC: Session persists after page refresh
→ Test: "should maintain session after browser refresh"
```

### 3. Determine Test Mode

Check if tests require authentication or extension:

```bash
# If criteria mention extension, auth, or logged-in state
if echo "$CRITERIA" | grep -qi "extension\|logged.in\|authenticated"; then
  echo "⚠️ Tests require Google Profile mode"
  echo "See: .claude/rules/playwright-modes.md"
fi
```

### 4. Generate Playwright Tests

Create test file at `frontend/e2e/{feature}.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';

test.describe('{Feature Name}', () => {
  // Generated from acceptance criteria

  test('should {criterion 1}', async ({ page }) => {
    // Navigate to feature
    await page.goto('/path');

    // Perform action
    await page.getByRole('button', { name: 'Action' }).click();

    // Assert outcome
    await expect(page.getByText('Expected')).toBeVisible();
  });

  test('should {criterion 2}', async ({ page }) => {
    // ...
  });
});
```

### 5. Validate Tests Work

Run the generated tests:

```bash
cd frontend
npx playwright test e2e/{feature}.spec.ts --reporter=list
```

If tests fail:
- Check selectors using `mcp__playwright__browser_snapshot`
- Adjust locators to match actual DOM
- Re-run until passing

### 6. Save to Regression Suite

Once tests pass:
1. Commit test file with feature
2. Tests run in CI on every PR
3. Suite grows over time

## Test Generation Patterns

### User Actions
```typescript
// Click
await page.getByRole('button', { name: 'Submit' }).click();

// Type
await page.getByLabel('Email').fill('user@example.com');

// Select
await page.getByRole('combobox').selectOption('value');

// Navigate
await page.goto('/dashboard');
```

### Assertions
```typescript
// Visible
await expect(page.getByText('Success')).toBeVisible();

// Hidden
await expect(page.getByText('Error')).not.toBeVisible();

// URL
await expect(page).toHaveURL('/dashboard');

// Count
await expect(page.getByRole('listitem')).toHaveCount(5);
```

### Waiting
```typescript
// Wait for navigation
await page.waitForURL('/new-page');

// Wait for element
await page.waitForSelector('.loaded');

// Wait for network idle
await page.waitForLoadState('networkidle');
```

## Output Format

```
═══════════════════════════════════════════════════════════════
  🧪 ACCEPTANCE TEST GENERATION
═══════════════════════════════════════════════════════════════

  Feature: {name}
  Source: {epic/prd/description}

  Acceptance Criteria Found:
  1. {criterion}
  2. {criterion}
  3. {criterion}

  Generated Tests:
  ✅ should {test 1}
  ✅ should {test 2}
  ❌ should {test 3} - failed, fixing...
  ✅ should {test 3} - fixed

  Test File: frontend/e2e/{feature}.spec.ts

═══════════════════════════════════════════════════════════════
  Results: {n}/{total} tests passing

  ➡️  Next steps:
     git add frontend/e2e/{feature}.spec.ts
     Commit with feature PR

  🔄 Run regression:
     cd frontend && npx playwright test

═══════════════════════════════════════════════════════════════
```

## Building Regression Suite Over Time

Each time you complete work:

1. **Feature complete** → Run `/testing:acceptance {feature}`
2. **Bug fixed** → Add test that reproduces the bug
3. **Before merge** → Run full regression: `npx playwright test`

Over time, the suite grows:
```
frontend/e2e/
  auth.spec.ts          # 5 tests
  dashboard.spec.ts     # 8 tests
  video-player.spec.ts  # 12 tests
  note-editor.spec.ts   # 15 tests
  ...
```

## Integration with PM Workflow

This skill is called during:
- `/pm:epic-verify` - Generates tests for completed epic
- After bug fixes - Generates regression test
- Before releases - Verifies all acceptance criteria
