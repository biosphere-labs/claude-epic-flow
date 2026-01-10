# Worktree Enforcement Rule

**CRITICAL**: All code changes MUST happen in worktrees, NEVER in the main checkout.

## Why This Matters

- The main checkout is for coordination, documentation, and commands only
- Code changes in main pollute the working directory
- Worktrees provide isolation for parallel work
- Uncommitted changes in main can cause merge conflicts

## What Belongs Where

### Main Checkout ✅
- `workflow/` directory (prds, epics, context, bugfixes)
- Documentation files (*.md)
- Running PM commands (`/pm:*`)
- Git operations (merging completed worktrees)

### Worktrees Only ✅
- Source code changes (src/, lib/, app/, etc.)
- Test file changes
- Any code file modifications

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
```

## When This Rule Applies

This rule applies when modifying code files. Common patterns:
- `**/*.{ts,tsx,js,jsx,py,go,rs,java}`
- `**/*.{css,scss,less}`
- Test files (`*.test.*`, `*.spec.*`)

**Exceptions** (can be modified in main checkout):
- `workflow/**/*` - PM system files
- `*.md` - Documentation
- Config files at root level

## Error Response

If about to modify code outside a worktree:

```
❌ Code changes must happen in a worktree

You're currently in the main checkout.
This file should be modified in a worktree instead.

To fix:
1. Run: /pm:epic-start <epic-name>
   This creates a worktree at .worktrees/epic/<name>

2. Or if resuming work on an existing epic:
   cd .worktrees/epic/<name>

Current location: {pwd}
Worktrees available: git worktree list
```

## Self-Check for Agents

Before using Edit/Write on code files:

1. Is the file a code file (not markdown/config)?
2. Are we in `.worktrees/`?
3. If code file and NOT in worktree, STOP and ask user to start the epic first

## Summary

| Location | Can Modify workflow/ | Can Modify Code |
|----------|---------------------|-----------------|
| Main checkout | ✅ Yes | ❌ No |
| Worktree (.worktrees/*) | ✅ Yes | ✅ Yes |

**When in doubt: If you're touching code, you should be in a worktree.**
