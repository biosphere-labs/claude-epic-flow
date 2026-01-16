---
allowed-tools: Bash, Read, Write, Edit, LS, Task, AskUserQuestion
---

# Epic One-Shot

Fast path for small, well-defined work that doesn't need task decomposition. Creates epic, worktree, and you implement directly.

## Usage
```
/pm:epic-oneshot <feature_name> [description]
```

## When to Use

**Use one-shot for:**
- Small features (1-3 files changed)
- Bug fixes that need epic tracking
- Config changes, documentation updates
- Quick refactors with clear scope
- Any work where decomposition is overhead

**Use full workflow instead (`epic-create → epic-decompose → epic-start`) for:**
- Multi-day features
- Work that benefits from parallel execution
- Features needing detailed task tracking

## Instructions

### 1. Preflight

```bash
# Check for project config
if [ ! -f ".claude/project.yaml" ]; then
  echo "⚠️ No project config - GitHub sync will be skipped"
else
  GITHUB_REPO=$(grep -A2 "^github:" .claude/project.yaml | grep "repo:" | sed 's/.*repo: *"\?\([^"]*\)"\?/\1/' | tr -d ' ')
fi

# Check if epic already exists
if [ -d "workflow/epics/$ARGUMENTS" ]; then
  echo "Epic already exists: workflow/epics/$ARGUMENTS"
  # Ask: Resume existing epic or create fresh?
fi

# Check for uncommitted changes
if [ -n "$(git status --porcelain)" ]; then
  echo "⚠️ Uncommitted changes detected"
  git status --short
  # Ask: Stash and continue?
fi
```

### 2. Create Minimal Epic

Create at `workflow/epics/$ARGUMENTS/epic.md`:

```markdown
---
name: $ARGUMENTS
status: in-progress
created: {ISO datetime from: date -u +"%Y-%m-%dT%H:%M:%SZ"}
updated: {ISO datetime}
progress: 0%
oneshot: true
github:
---

# Epic: $ARGUMENTS

## Overview
{Brief description - from user input or ask}

## Scope
{What's included - 2-5 bullet points}

## Success Criteria
- [ ] {criterion from description}

## Notes
{Any technical notes, decisions, or context}
```

### 3. Create Worktree

```bash
mkdir -p .worktrees/epic
git checkout staging
git pull origin staging

if ! git worktree list | grep -q ".worktrees/epic/$ARGUMENTS"; then
  git worktree add .worktrees/epic/$ARGUMENTS -b epic/$ARGUMENTS
  echo "✅ Created worktree: .worktrees/epic/$ARGUMENTS"
else
  echo "✅ Using existing worktree"
fi
```

### 4. Setup Dependencies (if monorepo)

```bash
WORKTREE_PATH=.worktrees/epic/$ARGUMENTS

# Only if these directories exist in main repo
if [ -d "backend/node_modules" ]; then
  cp -r backend/node_modules $WORKTREE_PATH/backend/ 2>/dev/null
  cp -r frontend/node_modules $WORKTREE_PATH/frontend/ 2>/dev/null
  cp -r packages/shared-types/node_modules $WORKTREE_PATH/packages/shared-types/ 2>/dev/null
  cp -r packages/shared-types/dist $WORKTREE_PATH/packages/shared-types/ 2>/dev/null
  cp backend/.env $WORKTREE_PATH/backend/.env 2>/dev/null
  cp frontend/.env $WORKTREE_PATH/frontend/.env 2>/dev/null
  echo "✅ Dependencies copied"
fi
```

### 5. Sync to GitHub (Optional)

```yaml
Skill:
  skill: "pm:epic-sync"
  args: "$ARGUMENTS"
```

### 6. Output

```
═══════════════════════════════════════════════════════════════
  ✅ ONE-SHOT EPIC READY
═══════════════════════════════════════════════════════════════

  Epic: workflow/epics/$ARGUMENTS/epic.md
  Worktree: .worktrees/epic/$ARGUMENTS
  Branch: epic/$ARGUMENTS
  GitHub: #{issue_num} ✓ (or "not synced")

  📍 Ready to implement - no task decomposition needed

  Work in worktree:
    cd .worktrees/epic/$ARGUMENTS

  Commit as you go:
    git add . && git commit -m "Epic $ARGUMENTS: {change}"

  When done:
    /pm:epic-verify $ARGUMENTS  ← Run tests
    /pm:epic-close $ARGUMENTS   ← Merge to staging

═══════════════════════════════════════════════════════════════
```

## Completing One-Shot Epics

One-shot epics skip directly to verify/close:

1. **Implement** - Work directly in the worktree, commit as you go
2. **Verify** - `/pm:epic-verify $ARGUMENTS` (runs tests)
3. **Close** - `/pm:epic-close $ARGUMENTS` (merges to staging)

Note: `epic-verify` and `epic-close` detect `oneshot: true` and skip task checks.

## Error Recovery

**Worktree exists but epic doesn't:**
```bash
git worktree remove .worktrees/epic/$ARGUMENTS
# Then run epic-oneshot again
```

**Want to convert to full workflow:**
```bash
# Just run decompose - it will create task files
/pm:epic-decompose $ARGUMENTS
# Then use epic-start as normal
```
