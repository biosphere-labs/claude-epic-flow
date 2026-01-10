---
allowed-tools: Bash, Read, Write, LS, Glob
---

# PRD Sync

Sync existing PRDs to GitHub as Backlog issues on the project Kanban board.

## Usage
```
/pm:prd-sync [prd_name]
```

**Arguments:**
- No argument: Lists all PRDs and their sync status
- PRD name: Syncs that specific PRD (e.g., `weave-transcript-extension`)

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

if [ -z "$GITHUB_REPO" ]; then
  echo "❌ GitHub repo not configured in .claude/project.yaml"
  exit 1
fi

# Optional: Project board settings (if configured)
PROJECT_NUMBER=$(grep "project_number:" .claude/project.yaml | sed 's/.*project_number: *//' | tr -d ' "')
PROJECT_OWNER=$(grep "project_owner:" .claude/project.yaml | sed 's/.*project_owner: *//' | tr -d ' "')
PROJECT_ID=$(grep "project_id:" .claude/project.yaml | sed 's/.*project_id: *//' | tr -d ' "')
STATUS_FIELD_ID=$(grep "status_field_id:" .claude/project.yaml | sed 's/.*status_field_id: *//' | tr -d ' "')
STATUS_BACKLOG=$(grep "Backlog_status_id:" .claude/project.yaml | sed 's/.*: *//' | tr -d ' "')
STATUS_TODO=$(grep "ToDo_status_id:" .claude/project.yaml | sed 's/.*: *//' | tr -d ' "')
```

## Instructions

### 0. Check Remote Repository

```bash
# GITHUB_REPO already loaded from project.yaml in Project Configuration section
remote_url=$(git remote get-url origin 2>/dev/null || echo "")
if [[ "$remote_url" == *"biosphere-labs/claude-epic-flow"* ]]; then
  echo "❌ Cannot sync to template repository"
  exit 1
fi

# Use GITHUB_REPO from project.yaml (already set above)
REPO="$GITHUB_REPO"
```

### 1. Handle No Arguments (List Mode)

If no `$ARGUMENTS` provided, list all PRDs with sync status:

```bash
echo "PRD Sync Status"
echo "==============="
echo ""

for prd in workflow/prds/*.md; do
  [ -f "$prd" ] || continue
  name=$(basename "$prd" .md)
  github=$(grep '^github:' "$prd" | sed 's/^github: *//')
  status=$(grep '^status:' "$prd" | sed 's/^status: *//')

  if [ -n "$github" ] && [ "$github" != "" ]; then
    issue_num=$(echo "$github" | grep -oE '[0-9]+$')
    echo "✅ $name (#$issue_num) - $status"
  else
    echo "⚪ $name - $status (not synced)"
  fi
done

echo ""
echo "To sync a PRD: /pm:prd-sync <name>"
```

### 2. Validate PRD Exists

```bash
PRD_FILE="workflow/prds/$ARGUMENTS.md"
if [ ! -f "$PRD_FILE" ]; then
  echo "❌ PRD not found: $PRD_FILE"
  echo "Available PRDs:"
  ls workflow/prds/*.md 2>/dev/null | xargs -n1 basename | sed 's/.md$//'
  exit 1
fi
```

### 3. Check if Already Synced

```bash
existing_url=$(grep '^github:' "$PRD_FILE" | sed 's/^github: *//')

# Also check if there's a corresponding epic
has_epic=false
if [ -d "workflow/epics/$ARGUMENTS" ]; then
  has_epic=true
  epic_url=$(grep '^github:' "workflow/epics/$ARGUMENTS/epic.md" 2>/dev/null | sed 's/^github: *//')
fi
```

### 4. Determine Sync Action

```bash
if [ -n "$existing_url" ] && [ "$existing_url" != "" ]; then
  # UPDATE existing issue
  issue_num=$(echo "$existing_url" | grep -oE '[0-9]+$')
  ACTION="update"
  echo "Updating existing issue #$issue_num..."
elif [ "$has_epic" = true ] && [ -n "$epic_url" ]; then
  # PRD was promoted to epic - use epic's issue
  echo "ℹ️ PRD has been promoted to epic"
  echo "Epic issue: $epic_url"
  echo "Use /pm:epic-sync $ARGUMENTS to sync the epic instead"
  exit 0
else
  # CREATE new issue
  ACTION="create"
  echo "Creating new issue for PRD..."
fi
```

### 5. Build Issue Body

```bash
# Get PRD metadata
prd_name=$(grep '^name:' "$PRD_FILE" | sed 's/^name: *//')
prd_desc=$(grep '^description:' "$PRD_FILE" | sed 's/^description: *//')
prd_status=$(grep '^status:' "$PRD_FILE" | sed 's/^status: *//')

# Strip frontmatter for body
sed '1,/^---$/d; 1,/^---$/d' "$PRD_FILE" > /tmp/prd-body.md

# Add footer
echo "" >> /tmp/prd-body.md
echo "---" >> /tmp/prd-body.md
echo "*This is a PRD (Product Requirements Document). When ready to implement, run \`/pm:epic-create $ARGUMENTS\`*" >> /tmp/prd-body.md
```

### 6. Create or Update Issue

```bash
if [ "$ACTION" = "create" ]; then
  issue_num=$(gh issue create \
    --repo "$REPO" \
    --title "PRD: $ARGUMENTS" \
    --body-file /tmp/prd-body.md \
    --label "prd,backlog" \
    --json number -q .number)
  echo "Created issue #$issue_num"
else
  gh issue edit "$issue_num" \
    --repo "$REPO" \
    --body-file /tmp/prd-body.md
  echo "Updated issue #$issue_num"
fi

repo=$(gh repo view --json nameWithOwner -q .nameWithOwner)
prd_url="https://github.com/$repo/issues/$issue_num"
```

### 7. Add to Project (Backlog column)

```bash
# Add to project
item_id=$(gh project item-add $PROJECT_NUMBER \
  --owner "$PROJECT_OWNER" \
  --url "$prd_url" \
  --format json 2>/dev/null | jq -r '.id' 2>/dev/null)

if [ -z "$item_id" ] || [ "$item_id" = "null" ]; then
  # Already exists, get ID
  item_id=$(gh project item-list $PROJECT_NUMBER \
    --owner "$PROJECT_OWNER" \
    --format json 2>/dev/null | \
    jq -r --arg url "$prd_url" '.items[] | select(.content.url == $url) | .id' 2>/dev/null)
fi

if [ -n "$item_id" ] && [ "$item_id" != "null" ]; then
  # Set to Backlog column (PRDs are always backlog until promoted)
  gh project item-edit \
    --project-id "$PROJECT_ID" \
    --id "$item_id" \
    --field-id "$STATUS_FIELD_ID" \
    --single-select-option-id "$STATUS_BACKLOG" 2>/dev/null
  echo "✅ Added to project → Backlog"
fi
```

### 8. Update PRD File

```bash
current_date=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Add/update github field
if grep -q '^github:' "$PRD_FILE"; then
  sed -i.bak "/^github:/c\github: $prd_url" "$PRD_FILE"
else
  sed -i.bak "/^status:/a\github: $prd_url" "$PRD_FILE"
fi

# Update timestamp if exists
if grep -q '^updated:' "$PRD_FILE"; then
  sed -i.bak "/^updated:/c\updated: $current_date" "$PRD_FILE"
fi

rm "${PRD_FILE}.bak" 2>/dev/null
```

### 9. Output

```
✅ PRD synced to GitHub
  - Issue: #${issue_num}
  - Title: PRD: $ARGUMENTS
  - Kanban: Backlog
  - URL: ${prd_url}

Next steps:
  - Review PRD: ${prd_url}
  - When ready: /pm:epic-create $ARGUMENTS
```

## Bulk Sync

To sync all unsynced PRDs:

```bash
for prd in workflow/prds/*.md; do
  name=$(basename "$prd" .md)
  github=$(grep '^github:' "$prd" | sed 's/^github: *//')
  if [ -z "$github" ] || [ "$github" = "" ]; then
    echo "Syncing: $name"
    # Run sync for this PRD
  fi
done
```

## Error Handling

- If PRD not found, show available PRDs
- If GitHub issue creation fails, don't update local file
- If PRD already has an epic, direct user to epic-sync instead
