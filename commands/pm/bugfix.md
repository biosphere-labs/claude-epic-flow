---
allowed-tools: Bash, Read, Write, Edit, LS, Task, Glob, Grep, mcp__episodic-memory__search, mcp__episodic-memory__read, mcp__playwright__browser_navigate, mcp__playwright__browser_snapshot, mcp__playwright__browser_click, mcp__playwright__browser_take_screenshot
---

# Bug Fix Workflow

All-in-one bug workflow: observe → gather context → analyze → fix → test.

**Creates a worktree** for isolated bug fixing with its own dev server.

## Usage
```
/pm:bugfix <observation-or-bug-list> [port]
```

**Parameters:**
- `$1` (input): Required. Can be:
  - A rough observation: "Login button doesn't work on mobile"
  - A file path to bug list: `bugs-to-fix.md`
  - Inline semicolon-separated descriptions: "Bug 1; Bug 2"
- `$2` (port): Optional. Backend port for dev server (default: 4000, frontend will be port+1)

## Examples
```bash
# Rough observation - will ask clarifying questions
/pm:bugfix "The video page is blank sometimes"

# Detailed observation
/pm:bugfix "When I click on a video in the library, sometimes it shows a blank page. Happens after navigating back."

# From a file (skips Phase 0)
/pm:bugfix bugs-to-fix.md

# Inline descriptions (skips Phase 0)
/pm:bugfix "Login button doesn't respond on mobile; Dashboard charts don't load for new users"

# With custom port
/pm:bugfix "Button bug" 5000
```

## Required Rules

**IMPORTANT:** Before executing, read:
- `.claude/rules/datetime.md` - For timestamps
- `.claude/rules/agent-coordination.md` - For parallel work
- `.claude/rules/worktree-operations.md` - For worktree setup
- `.claude/rules/episodic-memory-tools.md` - For memory searches

## Phase 0A: Prime Context (ALWAYS RUN FIRST)

Before any research or user interaction, load project context:

1. **Check context exists:**
   ```bash
   ls -la workflow/context/*.md 2>/dev/null | wc -l
   ```
   - If 0 files: Tell user "❌ No context found. Run /context:create first." and stop.

2. **Load context files in order:**
   - Read `workflow/context/project-overview.md` - Project understanding
   - Read `workflow/context/tech-context.md` - Technical stack
   - Read `workflow/context/progress.md` - Current status
   - Read `workflow/context/system-patterns.md` - Architecture patterns
   - Read `workflow/context/project-structure.md` - File organization

3. **Brief acknowledgment (don't be verbose):**
   ```
   🧠 Context loaded ({n} files). Project: {name}, Branch: {branch}
   ```

## Port Configuration

**Determine ports from parameters:**
- If `$2` is provided, use that as backend port (frontend = backend + 1)
- Otherwise default to 4000/4001

Example: `/pm:bugfix bugs.md 5000` → backend 5000, frontend 5001

Throughout this document, `{backend_port}` and `{frontend_port}` refer to these computed values.

## Instructions

You are fixing bugs from: **$ARGUMENTS**

### Phase 0: Observation & Context Gathering (Conditional)

**Determine if Phase 0 is needed:**
- If input is a file path (contains `/` or ends in `.md`): **SKIP to Phase 1**
- If input contains semicolons (multiple bugs): **SKIP to Phase 1**
- If input is a vague or single observation: **RUN Phase 0**

**Phase 0 Steps:**

#### 0.1 Ask Clarifying Questions

Before gathering context, ask the user clarifying questions to understand the bug better:

```
I'd like to understand this bug better before investigating.

1. **When does it happen?**
   - Always, sometimes, or only under specific conditions?
   - After certain actions (navigation, refresh, etc.)?

2. **What do you see?**
   - Blank page, error message, wrong data, etc.?
   - Any console errors visible?

3. **What should happen instead?**
   - Expected behavior?

4. **Any other context?**
   - Browser/device?
   - Recent changes you're aware of?
   - Does it happen in other similar places?
```

Wait for user response before continuing.

#### 0.2 Gather Context (Parallel)

Launch these searches in parallel using Task tool:

**Episodic Memory Search:**
```
Use mcp__episodic-memory__search with query derived from observation
Look for: past discussions about this feature, previous bugs, design decisions
```

**Git History Search:**
```yaml
Task:
  subagent_type: "Explore"
  prompt: "Find recent git commits (last 2 weeks) related to: {keywords from observation}.
           Identify what changed that might have caused this."
```

**Codebase Search:**
```yaml
Task:
  subagent_type: "Explore"
  prompt: "Find code related to: {feature from observation}.
           Identify files involved, component structure, error handling, edge cases."
```

#### 0.3 Synthesize & Create Bug Report

After context gathering:

1. **Correlate findings:**
   - Did episodic memory reveal past similar bugs?
   - Did recent commits touch related code?
   - What are probable root causes?

2. **Create bug report file:**
   ```bash
   TIMESTAMP=$(date +%Y%m%d-%H%M%S)
   mkdir -p workflow/bugfixes/observations
   ```

   Write to `workflow/bugfixes/observations/bug-$TIMESTAMP.md`:
   ```markdown
   ---
   id: BUG-$TIMESTAMP
   status: observed
   created: {ISO timestamp}
   priority: {inferred: critical|high|medium|low}
   ---

   # Bug: {synthesized title}

   ## Original Observation
   {$ARGUMENTS verbatim}

   ## User Clarifications
   {answers from 0.1}

   ## Description
   {Expanded description based on observation and context}

   ## Reproduction Steps
   1. {Step 1}
   2. {Step 2}
   3. Expected: ...
   4. Actual: ...

   ## Context Gathered

   ### Related Past Discussions
   {Summary of episodic memory findings}

   ### Recent Relevant Changes
   | Commit | Date | Description |
   |--------|------|-------------|
   | {hash} | {date} | {message} |

   ## Technical Analysis

   ### Likely Affected Files
   - {file}: {why relevant}

   ### Probable Root Causes
   1. **Most likely:** {cause} - {evidence}
   2. **Alternative:** {cause} - {evidence}

   ## Acceptance Criteria
   - [ ] {criterion 1}
   - [ ] {criterion 2}
   ```

3. **Show summary and confirm:**
   ```
   ═══════════════════════════════════════════════════════════════
     BUG REPORT CREATED
   ═══════════════════════════════════════════════════════════════

     Report: workflow/bugfixes/observations/bug-$TIMESTAMP.md
     Priority: {priority}
     Probable cause: {most likely cause}

     Context gathered:
     - Episodic memory: {n} related discussions
     - Git history: {n} relevant commits
     - Files identified: {n}

     Ready to proceed with fix?
     - Yes: Continue to create worktree and fix
     - No: Review/edit the report first
   ═══════════════════════════════════════════════════════════════
   ```

   Wait for user confirmation, then continue to Phase 1 using the created file.

---

### Phase 1: Parse Bug Descriptions

1. **Determine input type:**
   - If `$ARGUMENTS` looks like a file path (contains `/` or `.md`): Read the file
   - Otherwise: Parse as inline text

2. **Extract individual bugs:**

   From file (expected format):
   ```markdown
   ## Bug 1: {title}
   {description}

   ## Bug 2: {title}
   {description}
   ```

   From inline (semicolon separated):
   ```
   Bug 1 description; Bug 2 description
   ```

3. **Create bugfix session directory:**
   ```bash
   BUGFIX_ID=$(date +%Y%m%d-%H%M%S)
   mkdir -p workflow/bugfixes/$BUGFIX_ID
   echo "Session: $BUGFIX_ID"
   ```

4. **Create bug files and checkpoint:**
   For each bug, write to `workflow/bugfixes/$BUGFIX_ID/BUG-{n}.md`:
   ```markdown
   ---
   id: BUG-{n}
   status: open
   created: {ISO timestamp}
   priority: medium
   ---

   # Bug: {title}

   ## Description
   {parsed description}

   ## Analysis
   {to be filled}

   ## Fix Plan
   {to be filled}

   ## Files to Modify
   {to be filled}
   ```

   Write checkpoint to `workflow/bugfixes/$BUGFIX_ID/progress.md`:
   ```markdown
   ---
   session: $BUGFIX_ID
   phase: 1
   bugs_total: {n}
   bugs_fixed: 0
   iteration: 0
   worktree: ../bugfix-$BUGFIX_ID
   ports: [{backend_port}, {frontend_port}]
   ---
   ```

### Phase 2: Parallel Bug Analysis

**2.1 Research framework best practices** (use Context7 MCP if available):
```yaml
Task:
  subagent_type: "general-purpose"
  prompt: |
    Research best practices for fixing bugs related to: {summary of bugs}

    Tech stack:
    - Frontend: React, Vite, TypeScript
    - Backend: NestJS, TypeScript
    - Database: DynamoDB

    Use Context7 MCP if available to fetch:
    - Official patterns for error handling in this context
    - Common pitfalls for this type of bug
    - Any relevant library-specific guidance

    Return: Key insights that should guide the fix approach
```

**2.2 Spawn analysis agents:**
For each bug (up to 5 concurrent), use Task tool:

> Use Task tool with subagent_type "general-purpose"
> Prompt: "Analyze bug BUG-{n}. Description: {description}. Search codebase for relevant code. Identify root cause. Determine files to change. Create fix plan. Estimate effort (XS/S/M/L). Update the bug file at workflow/bugfixes/$BUGFIX_ID/BUG-{n}.md with Analysis, Fix Plan, and Files to Modify sections."

2. **Wait for all agents to complete.**

3. **Create analysis summary:**
   Write to `workflow/bugfixes/$BUGFIX_ID/analysis-summary.md`:
   ```markdown
   # Bug Analysis Summary

   | Bug | Root Cause | Effort | Files |
   |-----|------------|--------|-------|
   | BUG-001 | {cause} | S | 2 |
   | BUG-002 | {cause} | M | 4 |

   ## Parallel Groups
   Bugs touching different files can run together:
   - Group A: BUG-001, BUG-003 (different files)
   - Group B: BUG-002 (overlaps with BUG-001)
   ```

4. **Update checkpoint:** Edit progress.md to set `phase: 2`.

### Phase 3: Create Bugfix Worktree

1. **Check if already in a worktree:**
   ```bash
   git worktree list
   CURRENT_DIR=$(pwd)
   ```

2. **Create bugfix worktree:**
   ```bash
   # From main repo
   git checkout main
   git pull origin main
   git worktree add ../bugfix-$BUGFIX_ID -b bugfix/$BUGFIX_ID
   echo "Created worktree: ../bugfix-$BUGFIX_ID"
   ```

3. **Change to worktree:**
   ```bash
   cd ../bugfix-$BUGFIX_ID
   pwd
   ```

4. **Update checkpoint:** Edit progress.md to set `phase: 3`.

**Note:** Dev servers are NOT started automatically. They will be started before E2E testing in Phase 5.

### Phase 4: Parallel Bug Fixing

1. **Group bugs by file independence:**
   - Read analysis-summary.md
   - Bugs touching different files → parallel
   - Bugs touching same files → sequential within group

2. **Fix parallel group:**
   For each bug in the parallel group, use Task tool:

   > Use Task tool with subagent_type "general-purpose"
   > Prompt: "Working in worktree: ../bugfix-$BUGFIX_ID/. Fix bug BUG-{n}. Read workflow/bugfixes/$BUGFIX_ID/BUG-{n}.md for analysis and plan. Implement the fix. Update bug status to 'fixed'. Commit: fix({component}): {bug title}"

3. **Wait for group to complete, then start next group.**

4. **Update checkpoint:** Edit progress.md after each bug fixed.

### Phase 4.5: Write Regression Tests

**CRITICAL: Every bugfix MUST produce a regression test file.**

**CRITICAL: NO MOCKS IN E2E TESTS.** See `/rules/testing-philosophy.md`.

E2E regression tests must:
- Make real HTTP calls to the local backend (running against staging/test database)
- Use real authentication flows
- Interact with real services
- NEVER use jest.mock, mockResolvedValue, or any mocking

For each fixed bug, write a Playwright test that:
- Covers the **exact bug scenario** (same steps that triggered the bug)
- **Passes** with the fix in place
- Will **catch regressions** if the same bug is reintroduced
- Uses REAL backend calls (no mocks)

1. **Create regression test file:**
   ```bash
   # Naming convention: {feature}-{bug-id}.spec.ts
   # Location: frontend/e2e/regression/
   mkdir -p frontend/e2e/regression
   ```

2. **For each bug, generate test file:**

   > Use Task tool with subagent_type "general-purpose"
   > Prompt: |
   >   Write a Playwright regression test for BUG-{n}.
   >
   >   Bug details from: workflow/bugfixes/$BUGFIX_ID/BUG-{n}.md
   >
   >   Requirements:
   >   - File: frontend/e2e/regression/bug-{n}-{slug}.spec.ts
   >   - Test MUST reproduce the exact bug scenario
   >   - Include descriptive test name referencing the bug
   >   - Add comment linking to bug report
   >   - Test should be self-contained and repeatable
   >
   >   Example structure:
   >   ```typescript
   >   import { test, expect } from '@playwright/test';
   >
   >   // Regression test for BUG-{n}: {title}
   >   // See: workflow/bugfixes/$BUGFIX_ID/BUG-{n}.md
   >   test('should {expected behavior} - regression for BUG-{n}', async ({ page }) => {
   >     // Setup: Navigate to affected page
   >     // Action: Reproduce the bug scenario
   >     // Assert: Verify correct behavior (would have failed before fix)
   >   });
   >   ```
   >
   >   Commit: test(e2e): add regression test for BUG-{n}

3. **Verify regression test passes:**
   ```bash
   cd ../bugfix-$BUGFIX_ID
   npx playwright test frontend/e2e/regression/bug-{n}*.spec.ts
   ```

4. **Update bug file with test reference:**
   Add to each BUG-{n}.md:
   ```markdown
   ## Regression Test
   - File: `frontend/e2e/regression/bug-{n}-{slug}.spec.ts`
   - Added: {ISO timestamp}
   ```

### Phase 5: E2E Test Loop

1. **Start dev servers for testing:**
   ```bash
   cd ../bugfix-$BUGFIX_ID

   # Start dev servers if not already running
   if [ -f ./dev-worktree.sh ]; then
     ./dev-worktree.sh {backend_port} &
     sleep 10  # Wait for startup
   fi

   # Verify servers are running
   curl -s http://localhost:{frontend_port} > /dev/null && echo "Frontend OK"
   curl -s http://localhost:{backend_port}/health > /dev/null && echo "Backend OK"
   ```

2. **Run E2E tests against worktree server:**
   ```bash
   BASE_URL=http://localhost:{frontend_port} npx playwright test --reporter=list 2>&1 | tee workflow/bugfixes/$BUGFIX_ID/e2e-run-{iteration}.log
   ```

3. **Analyze failures:**
   - If all pass → proceed to Phase 6
   - If failures → analyze each

4. **For each failure, determine cause:**
   - Regression from our fix? → Revert that fix, re-analyze
   - New bug discovered? → Add to bug list
   - Test needs updating? → Update test

5. **Healing loop:**
   Use Task tool for each failure:

   > Use Task tool with subagent_type "general-purpose"
   > Prompt: "Working in worktree: ../bugfix-$BUGFIX_ID/. Test failure in {test-file}. Error: {message}. Related bugs fixed: {list}. Determine if this is: regression, new bug, or test issue. Fix appropriately and re-run the test."

6. **Iterate:**
   - Maximum 5 iterations
   - Update progress.md with iteration count
   - If max reached, report to user

7. **Stop dev servers when done:**
   ```bash
   pkill -f "dev-worktree" 2>/dev/null || true
   ```

### Phase 6: Final Verification

1. **Run full test suite:**
   ```bash
   cd ../bugfix-$BUGFIX_ID
   pnpm test 2>&1 | tail -20
   BASE_URL=http://localhost:{frontend_port} npx playwright test 2>&1 | tail -20
   ```

2. **Verify all bugs fixed:**
   Check each bug file - all should have `status: fixed`.

3. **Generate fix report:**
   Write to `workflow/bugfixes/$BUGFIX_ID/fix-report.md`:
   ```markdown
   ---
   completed: {ISO timestamp}
   status: {all-fixed|partial|failed}
   iterations: {count}
   worktree: ../bugfix-$BUGFIX_ID
   branch: bugfix/$BUGFIX_ID
   ---

   # Bug Fix Report: $BUGFIX_ID

   ## Summary
   - Bugs reported: {total}
   - Bugs fixed: {fixed}
   - Tests passing: {pass}/{total}

   ## Fixed Bugs
   | Bug | Title | Fix Summary | Commit |
   |-----|-------|-------------|--------|
   | BUG-001 | {title} | {summary} | abc123 |

   ## Remaining Issues
   - {any unfixed with reasons}

   ## Regression Tests Added
   | Bug | Test File | Description |
   |-----|-----------|-------------|
   | BUG-001 | `frontend/e2e/regression/bug-001-{slug}.spec.ts` | {what it tests} |

   These tests are now part of the permanent test suite and will catch regressions.
   ```

### Phase 7: Human Review Checkpoint

**STOP and alert the user:**

```
═══════════════════════════════════════════════════════════════
  BUG FIXES COMPLETE - AWAITING YOUR REVIEW
═══════════════════════════════════════════════════════════════

  All automated tests pass. Please verify the fixes manually:

  Dev Server: http://localhost:{frontend_port}
  Worktree:   ../bugfix-$BUGFIX_ID

  Bugs fixed:
  {list each bug with brief description}

  To review:
    1. Open http://localhost:{frontend_port} in your browser
    2. Test each bug fix scenario
    3. Check for regressions

  When done:
    ✓ Approved:  Say "approved" and I'll create the PR
    ✗ Issues:    Describe what's wrong and I'll fix it

═══════════════════════════════════════════════════════════════
```

**Wait for user response before proceeding.**

- If user says "approved": Continue to Phase 7.5
- If user reports issues: Return to Phase 4 to fix them

### Phase 7.5: Simplicity Review

Before finalizing, review fixes for over-engineering:

```yaml
Task:
  subagent_type: "general-purpose"
  prompt: |
    You are a code simplicity expert. Review the bug fixes for YAGNI violations:

    Changes made:
    {git diff --name-only from bugfix branch}

    For each changed file, check for:
    - Unnecessary abstractions added
    - Over-engineered error handling
    - Premature generalizations
    - Added complexity beyond what's needed to fix the bug

    Return:
    - List of simplicity concerns (if any)
    - For each concern: file, issue, simpler alternative
    - If fixes are appropriately minimal, say "Fixes are appropriately scoped"
```

**If concerns found:** Present each to user via AskUserQuestion with options:
- "Apply simpler approach" → Make the change
- "Keep as-is" → Document the decision

After review, continue to Phase 8.

### Phase 8: Commit and Create PR

1. **Ensure all changes committed:**
   ```bash
   cd ../bugfix-$BUGFIX_ID
   git status
   git log --oneline -5
   ```

2. **Push branch:**
   ```bash
   git push -u origin bugfix/$BUGFIX_ID
   ```

3. **Sync to GitHub:**

   ```yaml
   Skill:
     skill: "pm:bug-sync"
     args: "$BUGFIX_ID"
   ```

   This updates the GitHub issue and moves it to the correct Kanban column.

4. **Report to user:**
   ```
   ═══════════════════════════════════════════════════════════════
     ✅ BUGFIX SESSION COMPLETE
   ═══════════════════════════════════════════════════════════════

     Session: $BUGFIX_ID
     Fixed: {n}/{total} bugs
     Tests: {pass}/{total} passing
     Iterations: {count}
     GitHub: #{issue_num} ✓ Synced → In Review

     Worktree: ../bugfix-$BUGFIX_ID
     Branch: bugfix/$BUGFIX_ID
     Dev Server: http://localhost:{frontend_port}

     📍 Workflow position: Fixes verified, ready to merge

     ➡️  Next steps:
        gh pr create ... ← Create PR (recommended)
        git checkout staging && git merge bugfix/$BUGFIX_ID ← Direct merge

     🧹 Cleanup (after merge):
        git worktree remove ../bugfix-$BUGFIX_ID
        git branch -d bugfix/$BUGFIX_ID

     🔄 Related:
        /pm:status                    ← Overall workflow status

   ═══════════════════════════════════════════════════════════════
   ```

## Output Format

```
Bug Fix Session: $BUGFIX_ID

Phase 1: Parsing bugs...
  Found {n} bugs to fix

Phase 2: Analyzing (parallel)...
  BUG-001: {root-cause} (S)
  BUG-002: {root-cause} (M)

Phase 3: Creating worktree...
  Worktree: ../bugfix-$BUGFIX_ID
  Dev server: http://localhost:{frontend_port}

Phase 4: Fixing (parallel)...
  BUG-001: Fixed
  BUG-002: Fixed

Phase 5: E2E test loop...
  Iteration 1: 8/10 passing
  Iteration 2: 10/10 passing

Phase 6: Final verification...
  All tests passing

Phase 7: Human review checkpoint...
  [AWAITING YOUR REVIEW]

═══════════════════════════════════════════════════════════════
  BUG FIXES COMPLETE - AWAITING YOUR REVIEW
═══════════════════════════════════════════════════════════════
  ...

[After user approval]

Phase 8: Creating PR...
  Branch pushed, PR ready

Result: {n}/{total} BUGS FIXED
Report: workflow/bugfixes/$BUGFIX_ID/fix-report.md
```

## Error Recovery

If session ends mid-workflow:
1. Check `workflow/bugfixes/$BUGFIX_ID/progress.md` for last phase
2. The worktree still exists at `../bugfix-$BUGFIX_ID`
3. Resume by entering worktree and continuing from last phase
4. Use `/rewind` if needed

If max iterations reached:
```
Bug fix incomplete after 5 E2E iterations.

Remaining failures:
- {test}: {error}

Options:
1. Continue: manually fix and re-run tests
2. Partial PR: create PR with working fixes
3. Abandon: git worktree remove ../bugfix-$BUGFIX_ID && git branch -D bugfix/$BUGFIX_ID
```

If regression detected:
```
Regression detected: {test} broke after fixing BUG-{n}

Options:
1. Revert fix: git revert {commit}
2. Fix both: analyze interaction between changes
3. Accept: if behavior change is intentional
```

If worktree creation fails:
```
Cannot create worktree.

Try:
  git worktree prune
  git worktree list

Or check for existing worktree:
  ls ../bugfix-*
```
