---
allowed-tools: Bash, Read, Write, LS
---

# Epic Sync

Push epic to GitHub as a single issue with task checklist, attached to the project Kanban board.

## Usage
```
/pm:epic-sync <feature_name>
```

## Quick Check

```bash
# Verify project config exists
test -f .claude/project.yaml || echo "❌ No project config. Run: /pm:init"

# Verify epic exists
test -f workflow/epics/$ARGUMENTS/epic.md || echo "❌ Epic not found. Run: /pm:epic-create $ARGUMENTS"

# Count task files
ls workflow/epics/$ARGUMENTS/*.md 2>/dev/null | grep -v epic.md | wc -l
```

If no tasks found: "❌ No tasks to sync. Run: /pm:epic-decompose $ARGUMENTS"

## Instructions

### 0. Load Project Configuration

Read from `.claude/project.yaml` (see `/rules/project-config.md`):

```bash
# Check for project config
if [ ! -f ".claude/project.yaml" ]; then
  echo "❌ No project configuration found"
  echo "   Run: /pm:init to configure this project"
  exit 1
fi

# Extract GitHub repo (owner/repo format)
GITHUB_REPO=$(grep -A2 "^github:" .claude/project.yaml | grep "repo:" | sed 's/.*repo: *"\?\([^"]*\)"\?/\1/' | tr -d ' ')

if [ -z "$GITHUB_REPO" ]; then
  echo "❌ GitHub repo not configured in .claude/project.yaml"
  exit 1
fi

echo "Using repository: $GITHUB_REPO"

# Optional: Project board settings (if configured)
PROJECT_NUMBER=$(grep "project_number:" .claude/project.yaml | sed 's/.*project_number: *//' | tr -d ' "')
PROJECT_OWNER=$(grep "project_owner:" .claude/project.yaml | sed 's/.*project_owner: *//' | tr -d ' "')
PROJECT_ID=$(grep "project_id:" .claude/project.yaml | sed 's/.*project_id: *//' | tr -d ' "')
STATUS_FIELD_ID=$(grep "status_field_id:" .claude/project.yaml | sed 's/.*status_field_id: *//' | tr -d ' "')
```

### 1. Build Epic Issue Body

Create issue body with task checklist:

```bash
# Extract epic content without frontmatter
sed '1,/^---$/d; 1,/^---$/d' workflow/epics/$ARGUMENTS/epic.md > /tmp/epic-body-raw.md

# Remove the "## Tasks Created" section (we'll add our own checklist)
awk '/^## Tasks Created/,/^## [^T]|^$/{next} {print}' /tmp/epic-body-raw.md > /tmp/epic-intro.md
```

### 2. Generate Task Checklist

Build checklist from local task files:

```bash
cat > /tmp/task-checklist.md << 'EOF'

## Tasks

EOF

# For each task file, extract name and first line of description
for task_file in workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md; do
  [ -f "$task_file" ] || continue

  task_num=$(basename "$task_file" .md)
  task_name=$(grep '^name:' "$task_file" | sed 's/^name: *//')

  # Get first meaningful line from Description section
  task_desc=$(awk '/^## Description/{getline; while(/^$/) getline; print; exit}' "$task_file" | head -c 80)

  # Get status from frontmatter
  task_status=$(grep '^status:' "$task_file" | sed 's/^status: *//')

  if [ "$task_status" = "closed" ]; then
    echo "- [x] **${task_num}. ${task_name}** - ${task_desc}" >> /tmp/task-checklist.md
  else
    echo "- [ ] **${task_num}. ${task_name}** - ${task_desc}" >> /tmp/task-checklist.md
  fi
done

# Add task count
total=$(ls workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md 2>/dev/null | wc -l)
closed=$(grep -l '^status: closed' workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md 2>/dev/null | wc -l)
echo "" >> /tmp/task-checklist.md
echo "**Progress: ${closed}/${total} tasks complete**" >> /tmp/task-checklist.md
```

### 3. Calculate Progress and Determine Kanban Column

```bash
# Get progress from frontmatter (e.g., "73%" -> 73)
progress=$(grep '^progress:' workflow/epics/$ARGUMENTS/epic.md | sed 's/^progress: *//' | tr -d '%')
progress=${progress:-0}

# Get status from frontmatter
epic_status=$(grep '^status:' workflow/epics/$ARGUMENTS/epic.md | sed 's/^status: *//')

# Determine Kanban column based on progress/status
if [ "$epic_status" = "completed" ] || [ "$epic_status" = "closed" ] || [ "$progress" -eq 100 ]; then
  KANBAN_NAME="In Review"
elif [ "$progress" -gt 0 ]; then
  KANBAN_NAME="In Progress"
else
  KANBAN_NAME="ToDo"
fi

echo "Progress: ${progress}% → Kanban: ${KANBAN_NAME}"
```

### 4. Create or Update Issue

```bash
# Check if epic already has a GitHub URL
existing_url=$(grep '^github:' workflow/epics/$ARGUMENTS/epic.md | sed 's/^github: *//')

# Combine intro and checklist
cat /tmp/epic-intro.md /tmp/task-checklist.md > /tmp/epic-body.md

# Determine epic type
if grep -qi "bug\|fix\|issue\|problem\|error" /tmp/epic-body.md; then
  epic_type="bug"
else
  epic_type="feature"
fi

if [ -n "$existing_url" ]; then
  # UPDATE existing issue
  issue_num=$(echo "$existing_url" | grep -oE '[0-9]+$')
  gh issue edit "$issue_num" --repo "$GITHUB_REPO" --body-file /tmp/epic-body.md
  echo "Updated issue #${issue_num}"
  epic_number=$issue_num
else
  # CREATE new issue
  epic_number=$(gh issue create \
    --repo "$GITHUB_REPO" \
    --title "Epic: $ARGUMENTS" \
    --body-file /tmp/epic-body.md \
    --label "epic,$epic_type" \
    --json number -q .number)
  echo "Created issue #${epic_number}"
fi

epic_url="https://github.com/$GITHUB_REPO/issues/$epic_number"
```

### 5. Add to GitHub Project (if configured)

```bash
if [ -n "$PROJECT_NUMBER" ] && [ -n "$PROJECT_OWNER" ]; then
  # Add issue to project (if not already added)
  item_id=$(gh project item-add $PROJECT_NUMBER \
    --owner "$PROJECT_OWNER" \
    --url "$epic_url" \
    --format json 2>/dev/null | jq -r '.id' 2>/dev/null)

  if [ -z "$item_id" ] || [ "$item_id" = "null" ]; then
    # Item might already exist, get its ID
    item_id=$(gh project item-list $PROJECT_NUMBER \
      --owner "$PROJECT_OWNER" \
      --format json 2>/dev/null | \
      jq -r --arg url "$epic_url" '.items[] | select(.content.url == $url) | .id' 2>/dev/null)
  fi

  if [ -n "$item_id" ] && [ "$item_id" != "null" ] && [ -n "$PROJECT_ID" ] && [ -n "$STATUS_FIELD_ID" ]; then
    # Get status option ID based on Kanban name
    STATUS_OPTION_ID=$(grep "${KANBAN_NAME// /_}_status_id:" .claude/project.yaml | sed 's/.*: *//' | tr -d ' "')

    if [ -n "$STATUS_OPTION_ID" ]; then
      gh project item-edit \
        --project-id "$PROJECT_ID" \
        --id "$item_id" \
        --field-id "$STATUS_FIELD_ID" \
        --single-select-option-id "$STATUS_OPTION_ID" 2>/dev/null
    fi

    echo "✅ Added to project board → ${KANBAN_NAME}"
  else
    echo "⚠️ Could not add to project board (add manually)"
  fi
else
  echo "ℹ️  No project board configured (skipping)"
fi
```

### 6. Update Epic File

```bash
current_date=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Update epic frontmatter with GitHub URL (if new)
if [ -z "$existing_url" ]; then
  # Add github field if it doesn't exist
  if grep -q '^github:' workflow/epics/$ARGUMENTS/epic.md; then
    sed -i.bak "/^github:/c\github: $epic_url" workflow/epics/$ARGUMENTS/epic.md
  else
    sed -i.bak "/^---$/a\github: $epic_url" workflow/epics/$ARGUMENTS/epic.md
  fi
fi

# Update the updated timestamp
sed -i.bak "/^updated:/c\updated: $current_date" workflow/epics/$ARGUMENTS/epic.md
rm workflow/epics/$ARGUMENTS/epic.md.bak 2>/dev/null
```

### 7. Create/Update Mapping File

```bash
cat > workflow/epics/$ARGUMENTS/github-mapping.md << EOF
# GitHub Issue Mapping

Epic: #${epic_number} - ${epic_url}
Repository: ${GITHUB_REPO}

Local tasks: workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md

Synced: $(date -u +"%Y-%m-%dT%H:%M:%SZ")
EOF
```

### 8. Output

```
✅ Synced to GitHub
  - Epic: #${epic_number}
  - Tasks: ${total} items (${closed} complete)
  - Progress: ${progress}%
  - Kanban: ${KANBAN_NAME}
  - URL: ${epic_url}

Next: /pm:epic-start $ARGUMENTS
```

## Error Handling

- If project config missing, fail with instructions to run /pm:init
- If issue creation fails, report error and don't update local files
- If project add fails, issue is still created - add to project manually
- If re-sync fails, report but keep local state
