---
allowed-tools: Bash, Read, Write, LS, Task
---

# Epic Start

Create worktree, start development servers, and orchestrate task execution.

## Usage
```
/pm:epic-start <epic_name>
```

## What This Command Does

1. **Creates worktree** at `.worktrees/epic/{name}` with branch `epic/{name}`
2. **Updates epic status** to `in-progress`
3. **Syncs to GitHub** (status change reflected)
4. **Copies dependencies** (node_modules, .env files)
5. **Generates random ports** (avoids conflicts)
6. **Starts dev servers** using `dev-worktree.sh`
7. **Orchestrates task execution** based on decomposed tasks
8. **Auto-syncs epic** as tasks complete

## Preflight Check

```bash
# 1. Verify epic exists
if [ ! -f "workflow/epics/$ARGUMENTS/epic.md" ]; then
  echo "❌ Epic not found: $ARGUMENTS"
  echo "   Run: /pm:epic-create $ARGUMENTS"
  exit 1
fi

# 2. Verify tasks exist
task_count=$(ls workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md 2>/dev/null | wc -l)
if [ "$task_count" -eq 0 ]; then
  echo "❌ No tasks found for epic: $ARGUMENTS"
  echo "   Run: /pm:epic-decompose $ARGUMENTS"
  exit 1
fi
echo "✅ Found $task_count tasks"

# 3. Check GitHub sync (warning only)
github=$(grep "^github:" workflow/epics/$ARGUMENTS/epic.md 2>/dev/null | sed 's/^github: *//')
if [ -z "$github" ] || [ "$github" = "none" ]; then
  echo "⚠️  Epic not synced to GitHub - progress won't be tracked online"
  echo "   Optional: Run /pm:epic-sync $ARGUMENTS first"
  # Use gum confirm if available, otherwise continue with warning
  if command -v gum &> /dev/null; then
    gum confirm "Continue without GitHub sync?" || exit 1
  fi
fi

# 4. Check for uncommitted changes
if [ -n "$(git status --porcelain)" ]; then
  echo "⚠️  Uncommitted changes detected"
  git status --short
fi
```

## Instructions

### 1. Create Worktree

```bash
if [ -n "$(git status --porcelain)" ]; then
  echo "❌ You have uncommitted changes. Commit or stash them first."
  exit 1
fi

mkdir -p .worktrees/epic
if ! git worktree list | grep -q ".worktrees/epic/$ARGUMENTS"; then
  git checkout staging
  git pull origin staging
  git worktree add .worktrees/epic/$ARGUMENTS -b epic/$ARGUMENTS
  echo "✅ Created worktree: .worktrees/epic/$ARGUMENTS"
  WORKTREE_NEW=true
else
  echo "✅ Using existing worktree: .worktrees/epic/$ARGUMENTS"
  WORKTREE_NEW=false
fi

# Update epic status to in-progress
sed -i 's/^status: backlog/status: in-progress/' workflow/epics/$ARGUMENTS/epic.md
echo "✅ Epic status set to in-progress"
```

### 2. Sync to GitHub

Sync the status change to GitHub:

```yaml
Skill:
  skill: "pm:epic-sync"
  args: "$ARGUMENTS"
```

This updates the GitHub issue to reflect the epic is now in-progress.

### 3. Setup Worktree Dependencies

Copy gitignored files that the app needs to run (node_modules, .env, built artifacts):

```bash
WORKTREE_PATH=.worktrees/epic/$ARGUMENTS

# Only copy if worktree was just created or deps are missing
if [ "$WORKTREE_NEW" = true ] || [ ! -d "$WORKTREE_PATH/backend/node_modules" ]; then
  echo "Setting up worktree dependencies..."

  # Copy node_modules (faster than fresh pnpm install)
  cp -r backend/node_modules $WORKTREE_PATH/backend/
  cp -r frontend/node_modules $WORKTREE_PATH/frontend/
  cp -r packages/shared-types/node_modules $WORKTREE_PATH/packages/shared-types/

  # Copy shared-types dist (built artifacts are gitignored)
  cp -r packages/shared-types/dist $WORKTREE_PATH/packages/shared-types/

  # Copy .env files (required for app to run)
  cp backend/.env $WORKTREE_PATH/backend/.env
  cp frontend/.env $WORKTREE_PATH/frontend/.env

  echo "✅ Dependencies and config copied"
fi
```

### 4. Generate Random Ports

```bash
# Generate random ports in non-privileged range (10000-60000)
# Probability of collision is very low (~0.002%)
BACKEND_PORT=$((RANDOM % 50000 + 10000))
FRONTEND_PORT=$((BACKEND_PORT + 1))

echo "Using ports: Backend=$BACKEND_PORT, Frontend=$FRONTEND_PORT"
```

### 5. Start Development Servers

```bash
cd .worktrees/epic/$ARGUMENTS

if [ -f ./dev-worktree.sh ]; then
  ./dev-worktree.sh $BACKEND_PORT $FRONTEND_PORT &
  DEV_PID=$!
else
  (cd backend && PORT=$BACKEND_PORT pnpm run start:dev) &
  sleep 3
  (cd frontend && \
    VITE_API_GATEWAY_URL="http://localhost:$BACKEND_PORT" \
    VITE_API_BASE_URL="http://localhost:$BACKEND_PORT" \
    pnpm run dev -- --port $FRONTEND_PORT) &
fi
```

### 6. Analyze Tasks for Parallel Execution

Read task files and build execution plan:

```bash
# Identify which tasks can run in parallel
for task in workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md; do
  parallel=$(grep '^parallel:' "$task" | sed 's/parallel: *//')
  depends=$(grep '^depends_on:' "$task" | sed 's/depends_on: *//')
  status=$(grep '^status:' "$task" | sed 's/status: *//')

  # Categorize: ready (parallel, no deps) vs blocked (has deps)
done
```

### 7. Execute Tasks (Orchestration)

Launch parallel agents for ready tasks:

```yaml
Task:
  description: "Execute task {task_num} for epic $ARGUMENTS"
  subagent_type: "general-purpose"
  run_in_background: true
  prompt: |
    Execute task {task_num} in epic $ARGUMENTS

    Worktree: .worktrees/epic/$ARGUMENTS
    Task file: workflow/epics/$ARGUMENTS/{task_num}.md

    Requirements:
    1. Read full task details from the file
    2. Implement according to acceptance criteria
    3. Commit frequently: "Task {task_num}: {change}"
    4. When complete, update task status to "closed"
    5. Follow /rules/agent-coordination.md for parallel work

    On completion, the orchestrator will:
    - Update task status to closed
    - Auto-sync epic to GitHub via /pm:epic-sync
```

### 8. Monitor and Coordinate

As tasks complete:
- Detect completion (status: closed in frontmatter)
- **Automatically sync** via `/pm:epic-sync $ARGUMENTS` to update GitHub checkboxes
- Check if blocked tasks are now ready
- Launch newly-ready tasks

### 9. Update Execution Status

Create `workflow/epics/$ARGUMENTS/execution-status.md`:

```markdown
---
started: {datetime}
worktree: .worktrees/epic/$ARGUMENTS
branch: epic/$ARGUMENTS
ports:
  backend: {BACKEND_PORT}
  frontend: {FRONTEND_PORT}
---

# Execution Status

## Development Servers
- Frontend: http://localhost:{FRONTEND_PORT}
- Backend:  http://localhost:{BACKEND_PORT}

## Task Progress
- [ ] 001 - {name} - {status}
- [ ] 002 - {name} - {status}
...

## Agents Active
- Agent-1: Task 001 (in_progress)
- Agent-2: Task 002 (in_progress)
```

### 10. Output

```
✅ Epic orchestration started: $ARGUMENTS

Worktree: .worktrees/epic/$ARGUMENTS
Branch: epic/$ARGUMENTS
GitHub: Synced ✓

Development servers:
  Frontend: http://localhost:{FRONTEND_PORT}
  Backend:  http://localhost:{BACKEND_PORT}

Launching {n} parallel agents:
  - Task 001: {name} ✓ Started
  - Task 002: {name} ✓ Started
  - Task 003: {name} ⏸ Waiting (depends on 001)

As tasks complete:
  - Task status updated to closed
  - GitHub checkboxes updated via epic-sync

When all tasks done:
  /pm:epic-verify $ARGUMENTS  - Run tests (required)
  /pm:epic-close $ARGUMENTS   - Merge to staging
```

## Manual Override (Edge Cases)

If you need to manually update task status or sync:
```
# Edit task status directly in the markdown file
sed -i 's/^status: open/status: closed/' workflow/epics/$ARGUMENTS/001.md

# Then sync to GitHub
/pm:epic-sync $ARGUMENTS       - Update GitHub checkboxes
```

## Stopping the Epic

```bash
# Kill dev servers
lsof -ti:{BACKEND_PORT} | xargs kill -9 2>/dev/null
lsof -ti:{FRONTEND_PORT} | xargs kill -9 2>/dev/null

# Or if started with dev-worktree.sh:
pkill -f "dev-worktree"
```

## Error Handling

- Worktree creation fails: `git worktree prune` then retry
- Dependencies missing: Run setup step again or `pnpm install` in worktree
- Port collision (rare): Re-run command for new random ports
- Agent fails: Report error, continue with other tasks
- Task blocked: Wait for dependencies, report status
