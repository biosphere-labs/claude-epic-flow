# Development Workflow

This project uses CCPM (Claude Code Project Management) for structured development with parallel agent execution.

## Quick Reference

| Workflow | Commands |
|----------|----------|
| **New Feature** | `/brainstorm` → `/pm:plan-analyze` → `/pm:prd-new` → `/pm:epic-create` → `/pm:epic-decompose` → `/pm:epic-start-worktree` → `/pm:validate` → `/pm:epic-merge` |
| **Quick Feature** | `/brainstorm` → `/pm:plan-analyze` → `/pm:epic-decompose` (skip PRD for smaller features) |
| **Bug Fixing** | `/pm:bug-observe <observation>` → `/pm:bugfix <report>` (creates its own worktree) |
| **Quick Bug Fix** | `/pm:bugfix <inline-description>` (skip observation for simple bugs) |
| **Check Status** | `/pm:status`, `/pm:epic-show <name>` |

**Note:** Use `/pm:epic-start-worktree` (not `/pm:epic-start`) for parallel agent work with isolated testing.

---

## Workflow 1: Feature Development (Brainstorm to Ship)

### Step 1: Brainstorm
```
/brainstorm <your idea>
```

**What it does:**
- Researches competitors, similar solutions, UX patterns
- Asks clarifying questions in phases
- Outputs a design document to `workflow/brainstorms/brainstorm_*.md`

**Example:**
```
/brainstorm Add a spaced repetition system for flashcards based on video content
```

**Output:** `workflow/brainstorms/brainstorm_20251215-1430_spaced-repetition.md`

---

### Step 1.5: Analyze and Expand Plan (Optional but Recommended)
```
/pm:plan-analyze <brainstorm-file>
```

**What it does:**
- Runs the **spec-flow-analyzer** agent on your brainstorm
- Identifies all user flows and permutations
- Finds gaps, edge cases, and missing specifications
- Researches episodic memory for related past decisions
- Explores codebase architecture for integration points
- Creates a comprehensive implementation plan

**Example:**
```
/pm:plan-analyze workflow/brainstorms/brainstorm_20251215-1430_spaced-repetition.md
```

**Output:**
- `.claude/plans/spaced-repetition.md` - Full implementation plan
- Original brainstorm archived to `workflow/brainstorms/archive/`

**When to use:**
- Always recommended for medium/large features
- Skip for trivial features where brainstorm is sufficient

**Output includes:**
- User flow analysis with permutations matrix
- Edge cases and error states
- Gaps identified (critical, important, nice-to-have)
- Technical architecture breakdown
- Implementation approach and order
- Acceptance criteria organized by priority

---

### Step 2: Create PRD
```
/pm:prd-new <feature-name>
```

**What it does:**
- Launches a product requirements brainstorming session
- Asks clarifying questions about the feature
- Creates a comprehensive PRD with user stories, requirements, and success criteria
- Stores in `workflow/prds/<feature-name>.md`

**Example:**
```
/pm:prd-new spaced-repetition
```

**Tip:** Reference your `/brainstorm` output when answering the clarifying questions.

**Output:**
- `workflow/prds/spaced-repetition.md`

---

### Step 3: Create Epic from PRD
```
/pm:epic-create <feature-name>
```

**What it does:**
- Reads the existing PRD at `workflow/prds/<feature-name>.md`
- Converts requirements into a technical implementation plan
- Creates an Epic with architecture decisions and task breakdown preview
- Stores in `workflow/epics/<feature-name>/epic.md`

**Example:**
```
/pm:epic-create spaced-repetition
```

**Output:**
- `workflow/epics/spaced-repetition/epic.md`

---

### Step 4: Decompose into Tasks
```
/pm:epic-decompose <epic-name>
```

**What it does:**
- Breaks the epic into concrete, numbered tasks
- Sets `parallel: true/false` and `depends_on` for each task
- Each task has acceptance criteria and effort estimates

**Example:**
```
/pm:epic-decompose spaced-repetition
```

**Output:** `workflow/epics/spaced-repetition/001.md`, `002.md`, etc.

**Task file structure:**
```markdown
---
name: Create flashcard data model
status: open
parallel: true
depends_on: []
---

# Task: Create flashcard data model

## Acceptance Criteria
- [ ] Flashcard table with front/back fields
- [ ] Link to video timestamp
- [ ] Spaced repetition metadata fields

## Technical Details
...
```

---

### Step 5: Start Parallel Implementation
```
/pm:epic-start-worktree <epic-name>
```

**What it does:**
- Creates a git worktree: `.worktrees/epic/<name>/`
- Analyzes task dependencies
- Launches parallel agents for independent tasks
- Agents work in the SAME worktree on different files
- Commits directly with format: `Issue #X: <change>`

**Example:**
```
/pm:epic-start-worktree spaced-repetition
```

**What happens:**
```
Creating worktree: .worktrees/epic/spaced-repetition
Branch: epic/spaced-repetition

Launching agents:
  - Agent-1: Task 001 (Database) - Started
  - Agent-2: Task 002 (API) - Started
  - Agent-3: Task 003 (UI) - Waiting for 001, 002

Monitor: /pm:epic-status spaced-repetition
```

---

### Step 6: Validate Implementation
```
/pm:validate <epic-name>
```

**What it does:**
1. **Extracts acceptance criteria** from all task files
2. **Runs unit tests** - writes missing tests in parallel
3. **Runs E2E tests** - uses Playwright MCP for browser testing
4. **Heals failures** - attempts to auto-fix test issues
5. **Logs bugs** - creates bug files for unfixable issues
6. **Fixes bugs in parallel** - spawns agents to resolve
7. **Iterates** until all tests pass (max 3 iterations)

**Example:**
```
/pm:validate spaced-repetition
```

**Output:**
```
Phase 1: Extracting acceptance criteria...
  Found 12 criteria across 5 tasks

Phase 2: Running unit tests...
  45/48 passing
  Writing 3 missing tests...

Phase 3: E2E testing...
  Running 8 E2E tests...
  Healing 2 failures...

Phase 4: Bug fixing...
  Logged 1 bug, fixing...

Phase 5: Final validation...
  Unit: 48/48
  E2E: 8/8

Result: PASSED
Next: /pm:epic-merge spaced-repetition
```

---

### Step 7: Merge and Ship
```
/pm:epic-merge <epic-name>
```

**What it does:**
- Merges worktree branch to main
- Creates PR with summary
- Cleans up worktree

**Example:**
```
/pm:epic-merge spaced-repetition
```

---

## Workflow 2: Bug Fixing

For fixing known bugs without going through the full epic workflow.

### Step 1 (Optional): Create Detailed Bug Report from Observation
```
/pm:bug-observe <observation>
```

**What it does:**
- Takes a rough bug observation (symptoms, when it happens)
- Searches **episodic memory** for related past discussions
- Analyzes **git history** for recent related commits
- Searches **documents** for relevant specs
- Explores **codebase** for affected components
- Creates a detailed bug report with:
  - Root cause hypothesis
  - Recent changes that might be related
  - Acceptance criteria for the fix
  - Regression risks

**Example:**
```
/pm:bug-observe "Login button doesn't respond on mobile, have to double-tap"
```

**Output:**
- `workflow/bugfixes/observations/bug-{timestamp}.md` - Detailed report

**When to use:**
- Complex bugs where context would help
- Bugs that might have been encountered before
- When you want to understand WHY before fixing

---

### Step 2: Fix Bugs
```
/pm:bugfix <bug-list-or-description>
```

### From a Bug Observation Report
After running `/pm:bug-observe`:
```
/pm:bugfix workflow/bugfixes/observations/bug-20251215-143022.md
```

### From a Manual Bug List
Create a bug list file:
```markdown
## Bug 1: Login button unresponsive on mobile
Tapping the login button on iOS Safari does nothing.
Users have to double-tap.

## Bug 2: Dashboard charts empty for new users
New users see blank charts instead of "No data yet" message.
```

Then run:
```
/pm:bugfix bugs-to-fix.md
```

### Inline Description
```
/pm:bugfix "Login button unresponsive on mobile; Dashboard charts empty for new users"
```

### What Happens

1. **Parses bugs** into individual bug files
2. **Analyzes in parallel** - finds root causes, creates fix plans
3. **Creates worktree** (if not already in one)
4. **Fixes in parallel** - groups by file independence
5. **E2E test loop** - runs tests, heals failures, iterates
6. **Reports** - generates fix report with commits

**Output:**
```
Bug Fix Session: 20251215-143022

Phase 1: Parsing bugs...
  Found 2 bugs to fix

Phase 2: Analyzing (parallel)...
  BUG-001: Missing touch event handler (S)
  BUG-002: Empty state not rendered (S)

Phase 4: Fixing (parallel)...
  BUG-001: Fixed
  BUG-002: Fixed

Phase 5: E2E test loop...
  10/10 passing

Result: 2/2 BUGS FIXED
Report: workflow/bugfixes/20251215-143022/fix-report.md
```

---

## Utility Commands

### Check Overall Status
```
/pm:status
```
Shows all epics, PRDs, and their current state.

### View Epic Details
```
/pm:epic-show <name>
```
Shows epic tasks, progress, and dependencies.

### View What's Next
```
/pm:next
```
Shows the next task to work on based on dependencies.

### Sync with GitHub
```
/pm:epic-sync <name>
```
Syncs epic tasks with GitHub issues.

### Search Conversation History
```
/pm:search-history <query>
```
Searches past Claude Code conversations for context about what was discussed or decided. Uses the episodic memory MCP for semantic search.

**Examples:**
```bash
/pm:search-history spaced repetition implementation decisions
/pm:search-history login button bug fix attempts
/pm:search-history database schema design choices
```

---

## Knowledge Sources

The workflow uses multiple sources for context:

| Source | Location | Use For |
|--------|----------|---------|
| **Epics** | `workflow/epics/` | Current tasks, acceptance criteria |
| **PRDs** | `workflow/prds/` | Product requirements, scope |
| **Plans** | `.claude/plans/` | Implementation plans with user flows, gaps |
| **Brainstorms** | `workflow/brainstorms/brainstorm_*.md` | Research, design decisions |
| **Bug Observations** | `workflow/bugfixes/observations/` | Detailed bug reports with context |
| **Bug Sessions** | `workflow/bugfixes/<session>/` | Bug analysis, fix history |
| **Conversation History** | Episodic Memory MCP | Past discussions, decisions, attempts |

### When to Search History

Use `/pm:search-history` when:
- Starting work on a feature that was previously discussed
- Debugging an issue that may have been encountered before
- Looking for architectural decisions or rationale
- Finding out what was already tried for a problem
- Checking if a TODO was completed in a previous session

The file-based system (epics, PRDs, brainstorms) is the **source of truth** for current work.
Episodic memory is for **historical context** and **institutional knowledge**.

---

## Directory Structure

```
.claude/
├── brainstorms/          # Design documents from /brainstorm
│   ├── brainstorm_*.md   # Active brainstorms
│   └── archive/          # Processed brainstorms
├── plans/                # Full implementation plans (from plan-analyze)
│   └── <feature-name>.md
├── prds/                 # Product requirement docs
│   └── <feature-name>.md
├── epics/                # Epic directories
│   └── <epic-name>/
│       ├── epic.md       # Epic definition
│       ├── 001.md        # Task 1
│       ├── 002.md        # Task 2
│       ├── bugs/         # Bugs found during validation
│       └── validation-report.md
├── bugfixes/             # Bug fix sessions
│   ├── observations/     # Bug reports from bug-observe
│   │   └── bug-<timestamp>.md
│   └── <session-id>/     # Bug fix session logs
│       ├── BUG-001.md
│       └── fix-report.md
├── commands/pm/          # All /pm:* commands
├── agents/               # Agent definitions
├── rules/                # Execution rules
└── WORKFLOW.md           # This file
```

---

## Port Allocation

When running dev servers in worktrees:

| Location | Backend | Frontend |
|----------|---------|----------|
| Main repo | 3000 | 3001 |
| Epic worktree | 4000 | 4001 (default) |
| Bugfix worktree | 4000 | 4001 (default) |

Use `./dev-worktree.sh` for worktree instances:
```bash
./dev-worktree.sh        # Uses 4000/4001
./dev-worktree.sh 5000   # Uses 5000/5001
```

**Custom ports in workflows:**
```bash
/pm:validate epic-name 5000   # Uses ports 5000/5001
/pm:bugfix bugs.md 5000       # Uses ports 5000/5001
```

---

## Tips

1. **Always start with `/brainstorm`** - even for small features, it catches edge cases early

2. **Run `/pm:plan-analyze` for anything non-trivial** - the spec-flow-analyzer catches gaps you'll miss

3. **Check `/pm:status` often** - keeps you oriented on what's in progress

4. **Trust the parallel agents** - they commit directly, you review at merge time

5. **Use `/pm:bug-observe` for complex bugs** - context from history helps find root causes faster

6. **Use `/pm:bugfix` for quick fixes** - don't create a full epic for bug batches

7. **Validation is iterative** - it will loop up to 3 times to fix issues automatically

8. **Review before merge** - use `git diff main` in the worktree to review all changes
