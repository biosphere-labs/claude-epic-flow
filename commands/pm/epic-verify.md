---
description: Verify an epic before merging - runs unit tests, generates missing tests, then runs E2E tests. Required before epic-close.
allowed-tools: Bash, Read, Write, Edit, LS, Task, Glob, Grep, Skill
---

# Epic Verify

Comprehensive pre-merge verification. Orchestrates unit tests and E2E testing commands. **Required before epic-close.**

## Usage
```
/pm:epic-verify <epic_name>
```

## What This Command Does

1. **Detects changes** - What files changed in backend/frontend
2. **Runs unit tests** - Existing tests for changed areas
3. **Generates missing tests** - Calls `/testing:acceptance` to create E2E tests from acceptance criteria
4. **Runs E2E tests** - Calls `/testing:e2e` to run all Playwright tests
5. **Updates verification status** - Marks epic as verified (or not)
6. **Syncs to GitHub** - Calls `/pm:epic-sync` to update status

## Project Configuration

Read from `.claude/project.yaml`:

```bash
# Check for project config (optional for verification, required for sync)
if [ -f ".claude/project.yaml" ]; then
  GITHUB_REPO=$(grep -A2 "^github:" .claude/project.yaml | grep "repo:" | sed 's/.*repo: *"\?\([^"]*\)"\?/\1/' | tr -d ' ')
fi
```

## Quick Check

```bash
# Verify epic exists
test -d workflow/epics/$ARGUMENTS || echo "❌ Epic not found: $ARGUMENTS"

# Verify worktree exists (recommended but not required)
git worktree list | grep ".worktrees/epic/$ARGUMENTS" || echo "⚠️ No worktree found. Tests will run in main repo."
```

## Instructions

### Phase 1: Setup and Change Detection

#### 1.1 Determine Working Directory

```bash
# Check if worktree exists
if git worktree list | grep -q ".worktrees/epic/$ARGUMENTS"; then
  WORK_DIR=".worktrees/epic/$ARGUMENTS"
  echo "Using worktree: $WORK_DIR"
else
  WORK_DIR="."
  echo "Using main repository (no worktree found)"
fi
```

#### 1.2 Detect Changed Areas

```bash
cd $WORK_DIR

# Get changed files compared to staging
CHANGED_FILES=$(git diff --name-only staging...HEAD 2>/dev/null || git diff --name-only HEAD~10)

# Categorize changes
BACKEND_CHANGED=false
FRONTEND_CHANGED=false

echo "$CHANGED_FILES" | grep -q "^backend/" && BACKEND_CHANGED=true
echo "$CHANGED_FILES" | grep -q "^frontend/" && FRONTEND_CHANGED=true

echo "Changes detected:"
echo "  Backend: $BACKEND_CHANGED"
echo "  Frontend: $FRONTEND_CHANGED"

cd - > /dev/null
```

### Phase 2: Unit Tests

#### 2.1 Run Backend Tests

```bash
if [ "$BACKEND_CHANGED" = true ]; then
  echo "Running backend unit tests..."
  cd $WORK_DIR/backend
  pnpm test 2>&1 | tee /tmp/backend-test-results.txt
  BACKEND_TEST_EXIT=$?
  cd - > /dev/null

  if [ $BACKEND_TEST_EXIT -eq 0 ]; then
    echo "✅ Backend tests passed"
  else
    echo "❌ Backend tests failed"
  fi
fi
```

#### 2.2 Run Frontend Tests

```bash
if [ "$FRONTEND_CHANGED" = true ]; then
  echo "Running frontend unit tests..."
  cd $WORK_DIR/frontend
  pnpm test 2>&1 | tee /tmp/frontend-test-results.txt
  FRONTEND_TEST_EXIT=$?
  cd - > /dev/null

  if [ $FRONTEND_TEST_EXIT -eq 0 ]; then
    echo "✅ Frontend tests passed"
  else
    echo "❌ Frontend tests failed"
  fi
fi
```

### Phase 3: Generate Missing E2E Tests

Call `/testing:acceptance` to generate Playwright tests from the epic's acceptance criteria:

```yaml
Skill:
  skill: "testing:acceptance"
  args: "$ARGUMENTS"
```

This will:
- Read acceptance criteria from `workflow/epics/$ARGUMENTS/epic.md` and task files
- Generate Playwright test files at `frontend/e2e/{epic}.spec.ts`
- Validate the generated tests pass

### Phase 4: Run E2E Tests

Call `/testing:e2e` to run all Playwright tests:

```yaml
Skill:
  skill: "testing:e2e"
  args: "$ARGUMENTS"
```

This will:
- Determine appropriate mode (Chromium vs Google Profile)
- Run Playwright tests
- Report results

### Phase 5: Generate Verification Report

Create `workflow/epics/$ARGUMENTS/verification-report.md`:

```markdown
---
verified: {true|false}
verified_at: {ISO datetime}
unit_tests_passed: {true|false}
e2e_tests_passed: {true|false}
---

# Verification Report: $ARGUMENTS

**Date:** {timestamp}
**Working Directory:** {$WORK_DIR}

## Unit Tests

**Backend:** {PASS/FAIL/SKIPPED}
- Tests run: {count}
- Passed: {count}

**Frontend:** {PASS/FAIL/SKIPPED}
- Tests run: {count}
- Passed: {count}

## E2E Tests

**Result:** {PASS/FAIL}
- Tests run: {count}
- Passed: {count}

## Overall Result

**{VERIFIED - Ready to merge | NOT VERIFIED - Needs fixes}**

{If failed, list specific failures}
```

### Phase 6: Update Epic Status

```bash
current_date=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Add verified field to epic frontmatter if all tests passed
if [ "$ALL_TESTS_PASSED" = true ]; then
  # Update or add verified field
  if grep -q '^verified:' workflow/epics/$ARGUMENTS/epic.md; then
    sed -i "s/^verified:.*/verified: true/" workflow/epics/$ARGUMENTS/epic.md
  else
    sed -i "/^status:/a verified: true" workflow/epics/$ARGUMENTS/epic.md
  fi

  # Update verified_at
  if grep -q '^verified_at:' workflow/epics/$ARGUMENTS/epic.md; then
    sed -i "s/^verified_at:.*/verified_at: $current_date/" workflow/epics/$ARGUMENTS/epic.md
  else
    sed -i "/^verified:/a verified_at: $current_date" workflow/epics/$ARGUMENTS/epic.md
  fi
fi

# Update the updated timestamp
sed -i "s/^updated:.*/updated: $current_date/" workflow/epics/$ARGUMENTS/epic.md
```

### Phase 7: Sync to GitHub

Always sync verification status to GitHub:

```yaml
Skill:
  skill: "pm:epic-sync"
  args: "$ARGUMENTS"
```

### Phase 8: Output

**If all tests pass:**
```
═══════════════════════════════════════════════════════════════
  ✅ EPIC VERIFIED: $ARGUMENTS
═══════════════════════════════════════════════════════════════

  Unit Tests:
    Backend:  {PASS} ({n} tests)
    Frontend: {PASS} ({n} tests)

  E2E Tests: {PASS} ({n} tests)

  Report: workflow/epics/$ARGUMENTS/verification-report.md
  GitHub: Synced ✓

  ➡️  Ready to merge:
     /pm:epic-close $ARGUMENTS

═══════════════════════════════════════════════════════════════
```

**If tests fail:**
```
═══════════════════════════════════════════════════════════════
  ❌ VERIFICATION FAILED: $ARGUMENTS
═══════════════════════════════════════════════════════════════

  Unit Tests:
    Backend:  {PASS/FAIL}
    Frontend: {PASS/FAIL}

  E2E Tests: {FAIL}

  Failures:
    - {test name}: {error}
    - {test name}: {error}

  Report: workflow/epics/$ARGUMENTS/verification-report.md

  ➡️  Fix issues and re-verify:
     /pm:epic-verify $ARGUMENTS

═══════════════════════════════════════════════════════════════
```

## Error Handling

**Unit tests fail:**
- Report failures with file paths
- Continue to E2E tests (gather all failures)
- Final report shows all issues

**E2E tests fail:**
- Check if app is running
- Check Playwright configuration
- Take screenshots on failure

**No acceptance criteria found:**
- Warn but continue
- Skip test generation phase
- Run existing E2E tests only

## Important Notes

- This command is **required** before `/pm:epic-close`
- Creates verification-report.md used by epic-close
- Sets `verified: true` in epic frontmatter when all tests pass
- Always syncs to GitHub at the end
