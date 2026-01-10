---
description: Verify an epic before merging - runs unit tests, writes missing tests for new features, then E2E tests via Playwright MCP.
allowed-tools: Bash, Read, Write, Edit, LS, Task, Glob, Grep, mcp__playwright__browser_navigate, mcp__playwright__browser_snapshot, mcp__playwright__browser_click, mcp__playwright__browser_type, mcp__playwright__browser_fill_form, mcp__playwright__browser_take_screenshot, mcp__playwright__browser_console_messages, mcp__playwright__browser_network_requests, mcp__playwright__browser_select_option, mcp__playwright__browser_wait_for
---

# Epic Verify

Comprehensive pre-merge verification: unit tests, test coverage for new features, and E2E acceptance tests.

## Usage
```
/pm:epic-verify <epic_name> [port]
```

## What This Command Does

1. **Detects changes** - What files changed in backend/frontend
2. **Runs unit tests** - Existing tests for changed areas
3. **Writes missing tests** - Spawns agents for new features without tests
4. **E2E tests** - Playwright MCP for acceptance criteria
5. **Reports results** - Pass/fail with recommendations

## Quick Check

```bash
# Verify worktree exists
git worktree list | grep ".worktrees/epic/$ARGUMENTS" || echo "❌ No worktree. Run: /pm:epic-start $ARGUMENTS"

# Check epic docs exist
test -d workflow/epics/$ARGUMENTS || echo "❌ Epic not found: $ARGUMENTS"
```

## Instructions

### Phase 1: Setup and Change Detection

#### 1.1 Determine Ports

```bash
# Generate random ports to avoid conflicts
BACKEND_PORT=$((RANDOM % 50000 + 10000))
FRONTEND_PORT=$((BACKEND_PORT + 1))
FRONTEND_URL="http://localhost:$FRONTEND_PORT"

echo "Using ports: Backend=$BACKEND_PORT, Frontend=$FRONTEND_PORT"
```

#### 1.2 Setup Worktree Dependencies

```bash
WORKTREE_PATH=.worktrees/epic/$ARGUMENTS

# Check for missing dependencies
if [ ! -d "$WORKTREE_PATH/backend/node_modules" ]; then
  echo "Setting up worktree dependencies..."
  cp -r backend/node_modules $WORKTREE_PATH/backend/
  cp -r frontend/node_modules $WORKTREE_PATH/frontend/
  cp -r packages/shared-types/node_modules $WORKTREE_PATH/packages/shared-types/
  cp -r packages/shared-types/dist $WORKTREE_PATH/packages/shared-types/
fi

if [ ! -f "$WORKTREE_PATH/backend/.env" ]; then
  cp backend/.env $WORKTREE_PATH/backend/.env
  cp frontend/.env $WORKTREE_PATH/frontend/.env
fi
```

#### 1.3 Detect Changed Areas

```bash
cd $WORKTREE_PATH

# Get changed files compared to staging
CHANGED_FILES=$(git diff --name-only staging...HEAD)

# Categorize changes
BACKEND_CHANGED=false
FRONTEND_CHANGED=false
SHARED_CHANGED=false

echo "$CHANGED_FILES" | grep -q "^backend/" && BACKEND_CHANGED=true
echo "$CHANGED_FILES" | grep -q "^frontend/" && FRONTEND_CHANGED=true
echo "$CHANGED_FILES" | grep -q "^packages/shared-types/" && SHARED_CHANGED=true

echo "Changes detected:"
echo "  Backend: $BACKEND_CHANGED"
echo "  Frontend: $FRONTEND_CHANGED"
echo "  Shared types: $SHARED_CHANGED"

cd -
```

### Phase 2: Unit Tests

#### 2.1 Run Existing Tests

```bash
cd $WORKTREE_PATH

# Run backend tests if backend changed
if [ "$BACKEND_CHANGED" = true ]; then
  echo "Running backend unit tests..."
  cd backend
  pnpm test 2>&1 | tee /tmp/backend-test-results.txt
  BACKEND_TEST_EXIT=$?
  cd ..
fi

# Run frontend tests if frontend changed
if [ "$FRONTEND_CHANGED" = true ]; then
  echo "Running frontend unit tests..."
  cd frontend
  pnpm test 2>&1 | tee /tmp/frontend-test-results.txt
  FRONTEND_TEST_EXIT=$?
  cd ..
fi

cd -
```

#### 2.2 Identify New Features Without Tests

Read task files to identify new features, then check if tests exist:

```bash
# For each task, extract what was implemented
for task in workflow/epics/$ARGUMENTS/[0-9]*.md; do
  [ -f "$task" ] || continue
  task_name=$(grep '^name:' "$task" | sed 's/^name: *//')

  # Look for implementation files mentioned or created
  # Check if corresponding test files exist
done
```

Use Glob and Grep to:
1. Find new/modified source files in the epic branch
2. Check if corresponding `.test.ts`, `.spec.ts`, or `__tests__/` files exist
3. List files that need test coverage

#### 2.3 Write Missing Tests (Parallel)

For each new feature file without tests, spawn an agent:

```yaml
Task:
  description: "Write tests for {component}"
  subagent_type: "general-purpose"
  prompt: |
    Working in worktree: .worktrees/epic/$ARGUMENTS

    Write unit tests for: {source_file}

    Context from task file: workflow/epics/$ARGUMENTS/{task_num}.md

    Requirements:
    1. Read the source file to understand what it does
    2. Read the task file for acceptance criteria
    3. Check existing test patterns in the project:
       - Backend: backend/src/**/*.spec.ts
       - Frontend: frontend/src/**/*.test.tsx
    4. Write comprehensive tests covering:
       - Happy path
       - Edge cases from acceptance criteria
       - Error handling
    5. Run tests to verify they pass
    6. Commit: test({component}): add tests for {feature}
```

Launch multiple agents in parallel for independent test files.

#### 2.4 Re-run Tests After Writing

```bash
cd $WORKTREE_PATH

if [ "$BACKEND_CHANGED" = true ]; then
  echo "Re-running backend tests..."
  cd backend && pnpm test && cd ..
fi

if [ "$FRONTEND_CHANGED" = true ]; then
  echo "Re-running frontend tests..."
  cd frontend && pnpm test && cd ..
fi

cd -
```

### Phase 3: E2E Tests with Playwright MCP

#### 3.1 Start App

```bash
echo "Starting app in worktree..."
cd $WORKTREE_PATH

./dev-worktree.sh $BACKEND_PORT $FRONTEND_PORT &
APP_PID=$!

echo "Waiting for app to start..."
sleep 15

# Verify it's running
if curl -s "http://localhost:$FRONTEND_PORT" >/dev/null 2>&1; then
  echo "✅ App started: $FRONTEND_URL"
else
  echo "⚠️ App may still be starting..."
fi

cd -
```

#### 3.2 Gather Acceptance Criteria

Read epic and task files to extract acceptance criteria:

```bash
# Read epic definition
cat workflow/epics/$ARGUMENTS/epic.md

# Read all task files for acceptance criteria
for task in workflow/epics/$ARGUMENTS/[0-9]*.md; do
  echo "=== $(basename $task) ==="
  grep -A 20 "## Acceptance Criteria" "$task" || true
done
```

From these, extract:
- Feature descriptions → pages/flows to test
- Acceptance criteria → observable outcomes
- User stories → test scenarios

#### 3.3 Present Test Plan

Before executing, present the plan:

```markdown
## E2E Verification: $ARGUMENTS

**Testing against:** $FRONTEND_URL

Based on epic documentation, I'll verify:

### From Task #1: {task name}
1. {Observable outcome}
2. {Observable outcome}

### Smoke Tests
- Application loads without console errors
- Navigation works between main pages

**Proceed with these tests?** (yes/adjust/skip)
```

#### 3.4 Execute E2E Tests

For each criterion, use Playwright MCP:

**Navigation & Setup:**
```
browser_navigate → $FRONTEND_URL
browser_snapshot → Verify page loaded
browser_console_messages → Check for errors
```

**For each test scenario:**
```
1. browser_snapshot → Understand current page
2. browser_click/type/fill_form → Perform actions
3. browser_snapshot → Verify outcome
4. browser_take_screenshot → Evidence (on failure)
5. Record: PASS or FAIL with details
```

**Test execution principles:**
- Use accessibility tree from snapshot for selectors
- Retry flaky assertions up to 3 times
- Check console errors after each major action
- Take screenshots on failure

### Phase 4: Report Results

```markdown
## Verification Report: $ARGUMENTS

**Date:** {timestamp}
**Worktree:** .worktrees/epic/$ARGUMENTS

### Unit Tests

**Backend:** {PASS/FAIL}
- Tests run: {count}
- Passed: {count}
- New tests written: {count}

**Frontend:** {PASS/FAIL}
- Tests run: {count}
- Passed: {count}
- New tests written: {count}

### E2E Tests

**URL:** $FRONTEND_URL

✅ **PASS** {criterion}
✅ **PASS** {criterion}
❌ **FAIL** {criterion}
   - Expected: {expected}
   - Actual: {actual}

### Summary
- Unit tests: {pass}/{total}
- E2E tests: {pass}/{total}

### Recommendation
{READY TO MERGE / NEEDS FIXES}
```

### Phase 5: Next Steps

**If all tests pass:**
```
✅ Epic verified! Ready for merge.

Next: /pm:epic-close $ARGUMENTS
```

**If tests fail:**
```
❌ Some tests failed.

Failed unit tests:
- {list}

Failed E2E tests:
- {list}

Options:
1. Fix issues and re-verify: /pm:epic-verify $ARGUMENTS
2. Review failures and fix manually
3. Proceed anyway (not recommended): /pm:epic-close $ARGUMENTS
```

### Cleanup

```bash
# Offer to stop the app
echo "
App is still running on $FRONTEND_URL

Options:
1. Keep running for manual testing
2. Stop with: kill $APP_PID
3. Stop all worktree processes: pkill -f 'dev-worktree'
"
```

## Error Handling

**App won't start:**
```
❌ Could not start app in worktree

Check:
- Dependencies installed? cd .worktrees/epic/{name} && pnpm install
- Ports available? lsof -i:{port}
- Build errors? Check terminal output

Manual start:
  cd .worktrees/epic/{name} && ./dev-worktree.sh {port}
```

**Unit tests fail:**
```
❌ Unit tests failed

Review failures in:
- Backend: /tmp/backend-test-results.txt
- Frontend: /tmp/frontend-test-results.txt

Fix issues before proceeding with E2E tests.
```

**No acceptance criteria found:**
```
⚠️ No clear acceptance criteria in epic docs.

Options:
1. Add criteria to task files
2. Provide criteria now:
   - What should users be able to do?
   - What pages should work?
```

## Important Notes

- Tests execute against LIVE app, not mocks
- Unit tests run BEFORE E2E (fail fast)
- New features should have test coverage before merge
- Playwright MCP controls browser directly
- Results focus on business outcomes, not stack traces
