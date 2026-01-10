---
allowed-tools: Bash, Read, Write, LS
---

# Issue Close

Mark a task as complete and update the epic's checklist on GitHub.

## Usage
```
/pm:issue-close <epic_name> <task_number>
```

Example: `/pm:issue-close user-auth 003`

## Instructions

### 1. Find Local Task File

```bash
task_file="workflow/epics/$1/$2.md"
test -f "$task_file" || echo "❌ Task not found: $task_file"
```

### 2. Update Local Status

Get current datetime: `date -u +"%Y-%m-%dT%H:%M:%SZ"`

Update task file frontmatter:
```yaml
status: closed
updated: {current_datetime}
```

### 3. Update Epic Progress

Calculate new progress:
```bash
epic_dir="workflow/epics/$1"
total=$(ls $epic_dir/[0-9][0-9][0-9].md 2>/dev/null | wc -l)
closed=$(grep -l '^status: closed' $epic_dir/[0-9][0-9][0-9].md 2>/dev/null | wc -l)
progress=$((closed * 100 / total))

# Update epic.md frontmatter
sed -i.bak "s/^progress:.*/progress: ${progress}%/" $epic_dir/epic.md
rm $epic_dir/epic.md.bak 2>/dev/null
```

### 4. Sync to GitHub

Run issue-sync to update the epic's task checklist:
```
/pm:issue-sync $1
```

This will check off the completed task in the GitHub issue.

### 5. Output

```
✅ Closed task $2 in epic $1
  Local: Task marked complete
  Epic progress: {progress}% ({closed}/{total} tasks complete)

Next: /pm:issue-sync $1 (to update GitHub)
```

## Important Notes

- Tasks are tracked locally in 001.md, 002.md, etc.
- GitHub only has one issue per epic with a task checklist
- Use `/pm:issue-sync` to push progress to GitHub
