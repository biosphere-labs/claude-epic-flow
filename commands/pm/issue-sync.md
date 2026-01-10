---
allowed-tools: Bash, Read, Write, LS
---

# Issue Sync

Update epic's task checklist on GitHub to reflect local task status.

## Usage
```
/pm:issue-sync <epic_name>
```

## Quick Check

```bash
# Verify epic exists and has GitHub URL
test -f workflow/epics/$ARGUMENTS/epic.md || echo "❌ Epic not found"
grep -q '^github:' workflow/epics/$ARGUMENTS/epic.md || echo "❌ Epic not synced. Run: /pm:epic-sync $ARGUMENTS"
```

## Instructions

### 1. Check Remote Repository

```bash
remote_url=$(git remote get-url origin 2>/dev/null || echo "")
if [[ "$remote_url" == *"automazeio/ccpm"* ]]; then
  echo "❌ ERROR: Cannot sync to CCPM template repository!"
  exit 1
fi
```

### 2. Get Epic Issue Number

```bash
github_url=$(grep '^github:' workflow/epics/$ARGUMENTS/epic.md | sed 's/^github: *//')
issue_num=$(echo "$github_url" | grep -oE '[0-9]+$')

if [ -z "$issue_num" ]; then
  echo "❌ No GitHub issue found. Run: /pm:epic-sync $ARGUMENTS"
  exit 1
fi
```

### 3. Rebuild Task Checklist

Generate updated checklist from local task files:

```bash
# Get epic content without frontmatter, excluding old Tasks section
sed '1,/^---$/d; 1,/^---$/d' workflow/epics/$ARGUMENTS/epic.md > /tmp/epic-raw.md
awk '/^## Tasks Created/,/^## [^T]|^$/{next} /^## Tasks$/,/^## /{next} {print}' /tmp/epic-raw.md > /tmp/epic-intro.md

# Build fresh checklist
cat > /tmp/task-checklist.md << 'EOF'

## Tasks

EOF

for task_file in workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md; do
  [ -f "$task_file" ] || continue

  task_num=$(basename "$task_file" .md)
  task_name=$(grep '^name:' "$task_file" | sed 's/^name: *//')
  task_desc=$(awk '/^## Description/{getline; while(/^$/) getline; print; exit}' "$task_file" | head -c 80)
  task_status=$(grep '^status:' "$task_file" | sed 's/^status: *//')

  if [ "$task_status" = "closed" ]; then
    echo "- [x] **${task_num}. ${task_name}** - ${task_desc}" >> /tmp/task-checklist.md
  else
    echo "- [ ] **${task_num}. ${task_name}** - ${task_desc}" >> /tmp/task-checklist.md
  fi
done

# Add progress
total=$(ls workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md 2>/dev/null | wc -l)
closed=$(grep -l '^status: closed' workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md 2>/dev/null | wc -l)
echo "" >> /tmp/task-checklist.md
echo "**Progress: ${closed}/${total} tasks complete**" >> /tmp/task-checklist.md
echo "" >> /tmp/task-checklist.md
echo "_Last synced: $(date -u +"%Y-%m-%dT%H:%M:%SZ")_" >> /tmp/task-checklist.md

# Combine
cat /tmp/epic-intro.md /tmp/task-checklist.md > /tmp/epic-body.md
```

### 4. Update GitHub Issue

```bash
gh issue edit "$issue_num" --body-file /tmp/epic-body.md
```

### 5. Update Local Frontmatter

```bash
current_date=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
sed -i.bak "/^updated:/c\updated: $current_date" workflow/epics/$ARGUMENTS/epic.md
rm workflow/epics/$ARGUMENTS/epic.md.bak 2>/dev/null
```

### 6. Output

```
✅ Synced to GitHub Issue #${issue_num}
  - Progress: ${closed}/${total} tasks complete
  - URL: ${github_url}
```

## Marking Tasks Complete

To mark a task as complete locally (which will sync to GitHub):

```bash
# Update task frontmatter
sed -i 's/^status: open/status: closed/' workflow/epics/$ARGUMENTS/{task_num}.md

# Then re-sync
/pm:issue-sync $ARGUMENTS
```

## Error Handling

- If GitHub update fails, report but keep local state intact
- If epic not found, suggest `/pm:epic-sync`
