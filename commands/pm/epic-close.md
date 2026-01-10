---
allowed-tools: Bash, Read, Write, LS
---

# Epic Close

Complete an epic: verify tasks, merge branch, cleanup worktree, update status.

## Usage
```
/pm:epic-close <epic_name>
```

## Project Configuration

Read from `.claude/project.yaml`:

```bash
# Check for project config
if [ ! -f ".claude/project.yaml" ]; then
  echo "❌ No project configuration found"
  echo "   Run: /pm:init to configure this project"
  exit 1
fi

# Extract GitHub repo (owner/repo format)
GITHUB_REPO=$(grep -A2 "^github:" .claude/project.yaml | grep "repo:" | sed 's/.*repo: *"\?\([^"]*\)"\?/\1/' | tr -d ' ')

# Optional: Project board settings (if configured)
PROJECT_NUMBER=$(grep "project_number:" .claude/project.yaml | sed 's/.*project_number: *//' | tr -d ' "')
PROJECT_OWNER=$(grep "project_owner:" .claude/project.yaml | sed 's/.*project_owner: *//' | tr -d ' "')
```

## Instructions

### 1. Verify Epic Has Been Tested

Check that epic-verify has been run:

```bash
# Check for verified: true in epic frontmatter
verified=$(grep '^verified:' workflow/epics/$ARGUMENTS/epic.md 2>/dev/null | sed 's/verified: *//')

if [ "$verified" != "true" ]; then
  echo "❌ Epic has not been verified"
  echo ""
  echo "   You must run: /pm:epic-verify $ARGUMENTS"
  echo ""
  echo "   This runs tests and ensures the epic is ready for merge."
  echo "   Once verification passes, the epic can be closed."
  exit 1
fi

echo "✅ Epic verification: PASSED"

# Show verification timestamp
verified_at=$(grep '^verified_at:' workflow/epics/$ARGUMENTS/epic.md 2>/dev/null | sed 's/verified_at: *//')
if [ -n "$verified_at" ]; then
  echo "   Verified at: $verified_at"
fi
```

### 2. Verify All Tasks Complete

Check all task files in `workflow/epics/$ARGUMENTS/`:
- Verify all have `status: closed` in frontmatter
- If any open tasks found: "❌ Cannot close epic. Open tasks remain: {list}"

### 3. Check Worktree Status

```bash
# Check if worktree exists
if git worktree list | grep -q ".worktrees/epic/$ARGUMENTS"; then
  WORKTREE_EXISTS=true
  WORKTREE_PATH=$(git worktree list | grep ".worktrees/epic/$ARGUMENTS" | awk '{print $1}')
  echo "Found worktree: $WORKTREE_PATH"
else
  WORKTREE_EXISTS=false
  echo "No worktree found for epic"
fi
```

### 4. Verify Clean State (if worktree exists)

```bash
if [ "$WORKTREE_EXISTS" = true ]; then
  cd "$WORKTREE_PATH"

  # Check for uncommitted changes
  if [ -n "$(git status --porcelain)" ]; then
    echo "⚠️ Uncommitted changes in worktree:"
    git status --short
    echo ""
    echo "Options:"
    echo "  1. Commit changes first"
    echo "  2. Stash changes: git stash"
    echo "  3. Discard changes: git checkout -- ."
    # Ask user how to proceed
  fi

  cd - > /dev/null
fi
```

### 5. Merge Branch to Staging

```bash
# Return to main repo if in worktree
cd {main-repo-path}

# Ensure staging is up to date
git checkout staging
git pull origin staging

# Merge epic branch
echo "Merging epic/$ARGUMENTS to staging..."
git merge epic/$ARGUMENTS --no-ff -m "$(cat <<EOF
Merge epic: $ARGUMENTS

Completed tasks:
$(for task in workflow/epics/$ARGUMENTS/[0-9]*.md; do
  [ -f "$task" ] || continue
  name=$(grep '^name:' "$task" | sed 's/name: *//')
  num=$(basename "$task" .md)
  echo "- #$num: $name"
done)
EOF
)"

# If merge fails with conflicts
if [ $? -ne 0 ]; then
  echo "❌ Merge conflicts detected!"
  echo ""
  echo "Resolve conflicts manually, then run:"
  echo "  git add ."
  echo "  git commit"
  echo "  /pm:epic-close $ARGUMENTS  # to continue cleanup"
  exit 1
fi

# Push to staging
git push origin staging
echo "✅ Merged to staging and pushed"
```

### 6. Cleanup Worktree and Branch

```bash
if [ "$WORKTREE_EXISTS" = true ]; then
  # Remove worktree
  git worktree remove "$WORKTREE_PATH"
  echo "✅ Worktree removed: $WORKTREE_PATH"
fi

# Delete local branch
git branch -d epic/$ARGUMENTS 2>/dev/null || true

# Delete remote branch
git push origin --delete epic/$ARGUMENTS 2>/dev/null || true
echo "✅ Branch cleaned up: epic/$ARGUMENTS"
```

### 7. Update Epic Status

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Update epic.md frontmatter:
```yaml
status: completed
progress: 100%
updated: {current_datetime}
completed: {current_datetime}
```

### 8. Update PRD Status

If epic references a PRD, update its status to "complete".

### 9. Sync Final Status to GitHub

Update the GitHub issue with final task completion status before closing:

```bash
# Use GITHUB_REPO from project.yaml (loaded in Project Configuration section)
if [ -z "$GITHUB_REPO" ]; then
  echo "⚠️ Skipping GitHub sync - no GitHub repo configured"
else
  REPO="$GITHUB_REPO"

  # Check if epic has a GitHub issue
  existing_url=$(grep '^github:' workflow/epics/$ARGUMENTS/epic.md | sed 's/^github: *//')

  if [ -n "$existing_url" ]; then
    issue_num=$(echo "$existing_url" | grep -oE '[0-9]+$')

    # Build updated task checklist (all should be checked now)
    cat > /tmp/epic-final-body.md << 'INTRO'
# Epic Complete

All tasks have been completed and merged to staging.

## Tasks

INTRO

    for task_file in workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md; do
      [ -f "$task_file" ] || continue
      task_num=$(basename "$task_file" .md)
      task_name=$(grep '^name:' "$task_file" | sed 's/^name: *//')
      echo "- [x] **${task_num}. ${task_name}**" >> /tmp/epic-final-body.md
    done

    total=$(ls workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md 2>/dev/null | wc -l)
    echo "" >> /tmp/epic-final-body.md
    echo "**Progress: ${total}/${total} tasks complete** ✅" >> /tmp/epic-final-body.md
    echo "" >> /tmp/epic-final-body.md
    echo "Merged to staging: $(date -u +"%Y-%m-%d %H:%M UTC")" >> /tmp/epic-final-body.md

    # Update the issue
    gh issue edit "$issue_num" --body-file /tmp/epic-final-body.md
    echo "✅ Updated GitHub issue #${issue_num} with final status"
  fi
fi
```

### 10. Close GitHub Issues

```bash
# Close epic issue if exists
epic_issue=$(grep '^github:' workflow/epics/$ARGUMENTS/epic.md | grep -oE '[0-9]+$')
if [ -n "$epic_issue" ]; then
  gh issue close $epic_issue --comment "✅ Epic completed and merged to staging"
fi

# Close task issues
for task in workflow/epics/$ARGUMENTS/[0-9]*.md; do
  [ -f "$task" ] || continue
  task_issue=$(grep '^github:' "$task" | grep -oE '[0-9]+$')
  if [ -n "$task_issue" ]; then
    gh issue close $task_issue --comment "Completed in epic merge" 2>/dev/null || true
  fi
done
```

### 11. Archive Option

Ask user: "Archive completed epic? (yes/no)"

If yes:
```bash
mkdir -p workflow/epics/.archived/
mv workflow/epics/$ARGUMENTS workflow/epics/.archived/
echo "✅ Archived to workflow/epics/.archived/$ARGUMENTS"
```

### 12. Output

```
═══════════════════════════════════════════════════════════════
  EPIC COMPLETED: $ARGUMENTS
═══════════════════════════════════════════════════════════════

  Tasks completed: {count}
  Duration: {days}

  ✅ Merged to staging
  ✅ Worktree removed
  ✅ Branch deleted
  ✅ GitHub issue synced and closed
  {✅ Archived} (if applicable)

  Next steps:
    • Deploy staging to test
    • When ready for production: merge staging → main
    • Start new work: /pm:next

═══════════════════════════════════════════════════════════════
```

## Error Recovery

**Merge conflicts:**
- Resolve manually in the repo
- Run `git add . && git commit`
- Re-run `/pm:epic-close` to continue cleanup

**Worktree won't remove:**
```bash
git worktree remove --force .worktrees/epic/{name}
git worktree prune
```

**Branch already deleted:**
- Command handles this gracefully with `|| true`

## Important Notes

- **Verification required** - Must run `/pm:epic-verify` before close
- Only close epics with all tasks complete
- Merges to staging (not main) - deploy to production separately
- Preserves all data when archiving
