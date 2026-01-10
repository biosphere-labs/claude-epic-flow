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
# Verify epic exists
test -f workflow/epics/$ARGUMENTS/epic.md || echo "❌ Epic not found. Run: /pm:epic-create $ARGUMENTS"

# Count task files
ls workflow/epics/$ARGUMENTS/*.md 2>/dev/null | grep -v epic.md | wc -l
```

If no tasks found: "❌ No tasks to sync. Run: /pm:epic-decompose $ARGUMENTS"

## Project Configuration

```bash
# GitHub Project settings
PROJECT_NUMBER=8
PROJECT_OWNER="biosphere-labs"
PROJECT_ID="PVT_kwHOAKGL6M4BFEvm"
STATUS_FIELD_ID="PVTSSF_lAHOAKGL6M4BFEvmzg2hFR4"

# Status column option IDs
STATUS_TODO="66b38afa"
STATUS_IN_PROGRESS="47fc9ee4"
STATUS_IN_REVIEW="df73e18b"
STATUS_DONE="98236657"
```

## Instructions

### 0. Check Remote Repository

Follow `/rules/github-operations.md` to ensure we're not syncing to the CCPM template:

```bash
remote_url=$(git remote get-url origin 2>/dev/null || echo "")
if [[ "$remote_url" == *"automazeio/ccpm"* ]] || [[ "$remote_url" == *"automazeio/ccpm.git"* ]]; then
  echo "❌ ERROR: You're trying to sync with the CCPM template repository!"
  exit 1
fi
```

### 1. Detect GitHub Repository

```bash
remote_url=$(git remote get-url origin 2>/dev/null || echo "")
REPO=$(echo "$remote_url" | sed 's|.*github.com[:/]||' | sed 's|\.git$||')
[ -z "$REPO" ] && REPO="user/repo"
echo "Creating issue in repository: $REPO"
```

### 2. Build Epic Issue Body

Create issue body with task checklist:

```bash
# Extract epic content without frontmatter
sed '1,/^---$/d; 1,/^---$/d' workflow/epics/$ARGUMENTS/epic.md > /tmp/epic-body-raw.md

# Remove the "## Tasks Created" section (we'll add our own checklist)
awk '/^## Tasks Created/,/^## [^T]|^$/{next} {print}' /tmp/epic-body-raw.md > /tmp/epic-intro.md
```

### 3. Generate Task Checklist

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

### 4. Calculate Progress and Determine Kanban Column

```bash
# Get progress from frontmatter (e.g., "73%" -> 73)
progress=$(grep '^progress:' workflow/epics/$ARGUMENTS/epic.md | sed 's/^progress: *//' | tr -d '%')
progress=${progress:-0}

# Get status from frontmatter
epic_status=$(grep '^status:' workflow/epics/$ARGUMENTS/epic.md | sed 's/^status: *//')

# Determine Kanban column based on progress/status
# - ToDo: 0% progress
# - In Progress: >0% and <100%
# - In Review: closed/completed (100%)
if [ "$epic_status" = "completed" ] || [ "$epic_status" = "closed" ] || [ "$progress" -eq 100 ]; then
  KANBAN_STATUS="$STATUS_IN_REVIEW"
  KANBAN_NAME="In Review"
elif [ "$progress" -gt 0 ]; then
  KANBAN_STATUS="$STATUS_IN_PROGRESS"
  KANBAN_NAME="In Progress"
else
  KANBAN_STATUS="$STATUS_TODO"
  KANBAN_NAME="ToDo"
fi

echo "Progress: ${progress}% → Kanban: ${KANBAN_NAME}"
```

### 5. Create or Update Issue

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
  gh issue edit "$issue_num" --repo "$REPO" --body-file /tmp/epic-body.md
  echo "Updated issue #${issue_num}"
  epic_number=$issue_num
else
  # CREATE new issue
  epic_number=$(gh issue create \
    --repo "$REPO" \
    --title "Epic: $ARGUMENTS" \
    --body-file /tmp/epic-body.md \
    --label "epic,$epic_type" \
    --json number -q .number)
  echo "Created issue #${epic_number}"
fi
```

### 6. Add to GitHub Project and Set Kanban Column

```bash
repo=$(gh repo view --json nameWithOwner -q .nameWithOwner)
epic_url="https://github.com/$repo/issues/$epic_number"

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

if [ -n "$item_id" ] && [ "$item_id" != "null" ]; then
  # Set the Status field to correct Kanban column
  gh project item-edit \
    --project-id "$PROJECT_ID" \
    --id "$item_id" \
    --field-id "$STATUS_FIELD_ID" \
    --single-select-option-id "$KANBAN_STATUS" 2>/dev/null

  echo "✅ Added to project board → ${KANBAN_NAME}"
else
  echo "⚠️ Could not add to project board (add manually)"
fi
```

### 7. Update Epic File

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

### 8. Create/Update Mapping File

```bash
cat > workflow/epics/$ARGUMENTS/github-mapping.md << EOF
# GitHub Issue Mapping

Epic: #${epic_number} - ${epic_url}
Project: https://github.com/users/${PROJECT_OWNER}/projects/${PROJECT_NUMBER}
Kanban Status: ${KANBAN_NAME}

Local tasks: workflow/epics/$ARGUMENTS/[0-9][0-9][0-9].md

Synced: $(date -u +"%Y-%m-%dT%H:%M:%SZ")
EOF
```

### 9. Output

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

- If issue creation fails, report error and don't update local files
- If project add fails, issue is still created - add to project manually
- If re-sync fails, report but keep local state
