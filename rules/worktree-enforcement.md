# Worktree Enforcement Rule

**CRITICAL**: All code changes MUST happen in worktrees, NEVER in the main checkout (staging branch).

## Why This Matters

- The main checkout is for coordination, documentation, and commands only
- Code changes in staging pollute the working directory
- Worktrees provide isolation for parallel work
- Uncommitted changes in main can cause merge conflicts

## What Belongs Where

### Main Checkout (staging branch) ✅
- `.claude/` directory (commands, rules, prds, epics, context)
- Documentation files
- Running PM commands (`/pm:*`)
- Git operations (merging completed worktrees)

### Worktrees Only ✅
- `frontend/` code changes
- `backend/` code changes
- `packages/` code changes
- Test file changes
- Any `.ts`, `.tsx`, `.js`, `.jsx`, `.css`, `.scss` modifications

## Enforcement Check

Before modifying any code file, verify you're in a worktree:

```bash
# Check if current directory is a worktree
is_worktree() {
  if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
    git_dir=$(git rev-parse --git-dir)
    if [[ "$git_dir" == *".git/worktrees/"* ]] || [[ "$git_dir" == "../.git/worktrees/"* ]]; then
      return 0  # Is a worktree
    fi
  fi
  return 1  # Not a worktree
}

# Alternative: check if in .worktrees directory
is_in_worktree_dir() {
  pwd | grep -q "\.worktrees/" && return 0
  return 1
}
```

## When This Rule Applies

This rule applies when ANY of these patterns are about to be modified:

```
frontend/**/*.{ts,tsx,js,jsx,css,scss,json}
backend/**/*.{ts,js,json}
packages/**/*.{ts,js,json}
*.test.{ts,tsx,js}
*.spec.{ts,tsx,js}
```

**Exceptions** (can be modified in main checkout):
- `.claude/**/*` - PM system files
- `*.md` - Documentation
- `.env*` - Environment files (though be careful)
- `package.json` at root - Workspace config only

## Error Response

If about to modify code outside a worktree:

```
❌ Code changes must happen in a worktree

You're currently in the main checkout (staging branch).
This file should be modified in a worktree instead.

To fix:
1. Run: /pm:epic-start <epic-name>
   This creates a worktree at .worktrees/epic/<name>

2. Or if resuming work on an existing epic:
   cd .worktrees/epic/<name>

3. Then retry your changes there.

Current location: {pwd}
Worktrees available:
{git worktree list}
```

## Resuming Work on an Epic

If an epic worktree already exists:

```bash
# List existing worktrees
git worktree list

# Navigate to epic worktree
cd .worktrees/epic/<epic-name>

# Verify you're in the worktree
pwd  # Should show .worktrees/epic/<name>

# Continue work
```

## Creating a New Worktree

If no worktree exists for the work:

```bash
# Use the PM command (preferred)
/pm:epic-start <epic-name>

# Or manually
git worktree add .worktrees/epic/<name> -b epic/<name>
cd .worktrees/epic/<name>
```

## Integration with PM Commands

Commands that might trigger code changes should check:

- `/pm:bugfix` - Creates worktree automatically
- `/pm:epic-start` - Creates worktree automatically
- Task execution agents - Should verify worktree before editing

## Self-Check for Agents

Before using Edit/Write on code files:

1. Check file path - is it in `frontend/`, `backend/`, or `packages/`?
2. If yes, check current directory - are we in `.worktrees/`?
3. If not in worktree, STOP and ask user to start the epic first

```bash
# Quick check
if [[ "$FILE_PATH" =~ ^(frontend|backend|packages)/ ]]; then
  if ! pwd | grep -q "\.worktrees/"; then
    echo "❌ Must be in worktree to modify $FILE_PATH"
    exit 1
  fi
fi
```

## Summary

| Location | Can Modify .claude/ | Can Modify Code |
|----------|--------------------|-----------------|
| Main checkout (staging) | ✅ Yes | ❌ No |
| Worktree (.worktrees/*) | ✅ Yes | ✅ Yes |

**When in doubt: If you're touching code, you should be in a worktree.**
