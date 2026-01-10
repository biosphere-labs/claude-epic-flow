---
allowed-tools: Bash, Read, Write, LS, Glob
---

# Bug Sync

Sync bug observations or bug fix sessions to GitHub as issues, attached to the project Kanban board.

## Usage
```
/pm:bug-sync [bug-id-or-path]
```

**Arguments:**
- No argument: Syncs the most recent bug observation or active bugfix session
- Bug ID: e.g., `20260108-104032` - syncs that specific bugfix session
- File path: e.g., `workflow/bugfixes/observations/bug-20260108-230000.md` - syncs that observation

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

# Status column option IDs
STATUS_TODO=$(grep "ToDo_status_id:" .claude/project.yaml | sed 's/.*: *//' | tr -d ' "')
STATUS_IN_PROGRESS=$(grep "In_Progress_status_id:" .claude/project.yaml | sed 's/.*: *//' | tr -d ' "')
STATUS_IN_REVIEW=$(grep "In_Review_status_id:" .claude/project.yaml | sed 's/.*: *//' | tr -d ' "')
STATUS_DONE=$(grep "Done_status_id:" .claude/project.yaml | sed 's/.*: *//' | tr -d ' "')
```

## Instructions

### 0. Check Remote Repository

Follow `/rules/github-operations.md`:

```bash
# GITHUB_REPO already loaded from project.yaml in Project Configuration section
remote_url=$(git remote get-url origin 2>/dev/null || echo "")
if [[ "$remote_url" == *"automazeio/ccpm"* ]] || [[ "$remote_url" == *"automazeio/ccpm.git"* ]]; then
  echo "❌ ERROR: You're trying to sync with the CCPM template repository!"
  exit 1
fi

# Use GITHUB_REPO from project.yaml (already set above)
REPO="$GITHUB_REPO"
```

### 1. Determine Bug Source

```bash
ARG="$ARGUMENTS"

if [ -z "$ARG" ]; then
  # No argument - find most recent
  # Check for active bugfix session first
  LATEST_SESSION=$(ls -td workflow/bugfixes/20*/ 2>/dev/null | head -1)
  # Then check observations
  LATEST_OBS=$(ls -t workflow/bugfixes/observations/*.md 2>/dev/null | head -1)

  if [ -n "$LATEST_SESSION" ]; then
    BUG_SOURCE="session"
    BUG_PATH="$LATEST_SESSION"
    SESSION_ID=$(basename "$LATEST_SESSION")
  elif [ -n "$LATEST_OBS" ]; then
    BUG_SOURCE="observation"
    BUG_PATH="$LATEST_OBS"
  else
    echo "❌ No bugs found. Run /pm:bug-observe or /pm:bugfix first."
    exit 1
  fi
elif [ -f "$ARG" ]; then
  # Direct file path
  if [[ "$ARG" == *"/observations/"* ]]; then
    BUG_SOURCE="observation"
    BUG_PATH="$ARG"
  else
    BUG_SOURCE="session"
    BUG_PATH="$(dirname "$ARG")"
    SESSION_ID=$(basename "$BUG_PATH")
  fi
elif [ -d "workflow/bugfixes/$ARG" ]; then
  # Session ID provided
  BUG_SOURCE="session"
  BUG_PATH="workflow/bugfixes/$ARG"
  SESSION_ID="$ARG"
else
  echo "❌ Bug not found: $ARG"
  echo "Try: /pm:bug-sync 20260108-104032"
  exit 1
fi

echo "Syncing $BUG_SOURCE: $BUG_PATH"
```

### 2. Build Issue Content

**For Observations:**
```bash
if [ "$BUG_SOURCE" = "observation" ]; then
  # Extract frontmatter
  bug_id=$(grep '^id:' "$BUG_PATH" | sed 's/^id: *//')
  bug_status=$(grep '^status:' "$BUG_PATH" | sed 's/^status: *//')
  bug_priority=$(grep '^priority:' "$BUG_PATH" | sed 's/^priority: *//')

  # Extract title (first H1)
  bug_title=$(grep '^# Bug:' "$BUG_PATH" | sed 's/^# Bug: *//')
  [ -z "$bug_title" ] && bug_title="Bug: $bug_id"

  # Strip frontmatter for body
  sed '1,/^---$/d; 1,/^---$/d' "$BUG_PATH" > /tmp/bug-body.md

  # Kanban: observations are ToDo
  KANBAN_STATUS="$STATUS_TODO"
  KANBAN_NAME="ToDo"
fi
```

**For Sessions (completed bugfixes):**
```bash
if [ "$BUG_SOURCE" = "session" ]; then
  # Check for fix report (indicates completion)
  if [ -f "$BUG_PATH/fix-report.md" ]; then
    # Use fix report
    bug_status=$(grep '^status:' "$BUG_PATH/fix-report.md" | sed 's/^status: *//')
    bug_title="Bugfix: $SESSION_ID"

    # Build body from fix report
    sed '1,/^---$/d; 1,/^---$/d' "$BUG_PATH/fix-report.md" > /tmp/bug-body.md

    # Kanban: completed fixes go to In Review
    if [ "$bug_status" = "all-fixed" ]; then
      KANBAN_STATUS="$STATUS_IN_REVIEW"
      KANBAN_NAME="In Review"
    else
      KANBAN_STATUS="$STATUS_IN_PROGRESS"
      KANBAN_NAME="In Progress"
    fi
  elif [ -f "$BUG_PATH/progress.md" ]; then
    # In-progress session
    bugs_fixed=$(grep '^bugs_fixed:' "$BUG_PATH/progress.md" | sed 's/^bugs_fixed: *//')
    bugs_total=$(grep '^bugs_total:' "$BUG_PATH/progress.md" | sed 's/^bugs_total: *//')
    bug_status="in-progress"
    bug_title="Bugfix: $SESSION_ID"

    # Build body from BUG files
    echo "# Bugfix Session: $SESSION_ID" > /tmp/bug-body.md
    echo "" >> /tmp/bug-body.md
    echo "**Progress: ${bugs_fixed:-0}/${bugs_total:-?} bugs fixed**" >> /tmp/bug-body.md
    echo "" >> /tmp/bug-body.md
    echo "## Bugs" >> /tmp/bug-body.md

    for bug_file in "$BUG_PATH"/BUG-*.md; do
      [ -f "$bug_file" ] || continue
      name=$(grep '^# Bug:' "$bug_file" | sed 's/^# Bug: *//')
      status=$(grep '^status:' "$bug_file" | sed 's/^status: *//')
      if [ "$status" = "fixed" ]; then
        echo "- [x] $name" >> /tmp/bug-body.md
      else
        echo "- [ ] $name" >> /tmp/bug-body.md
      fi
    done

    KANBAN_STATUS="$STATUS_IN_PROGRESS"
    KANBAN_NAME="In Progress"
  else
    # Just analysis
    bug_title="Bug Analysis: $SESSION_ID"
    if [ -f "$BUG_PATH/analysis-summary.md" ]; then
      cp "$BUG_PATH/analysis-summary.md" /tmp/bug-body.md
    else
      echo "Bug analysis session $SESSION_ID" > /tmp/bug-body.md
    fi
    KANBAN_STATUS="$STATUS_TODO"
    KANBAN_NAME="ToDo"
  fi
fi
```

### 3. Check for Existing GitHub Issue

```bash
# Check if already synced (look for github field in any file)
existing_url=""
for f in "$BUG_PATH"/*.md "$BUG_PATH" 2>/dev/null; do
  [ -f "$f" ] || continue
  url=$(grep '^github:' "$f" 2>/dev/null | sed 's/^github: *//')
  if [ -n "$url" ]; then
    existing_url="$url"
    break
  fi
done
```

### 4. Create or Update Issue

```bash
# Set labels
labels="bug"
if [ -n "$bug_priority" ]; then
  case "$bug_priority" in
    critical) labels="$labels,priority:critical" ;;
    high) labels="$labels,priority:high" ;;
    medium) labels="$labels,priority:medium" ;;
    low) labels="$labels,priority:low" ;;
  esac
fi

if [ -n "$existing_url" ]; then
  # UPDATE existing issue
  issue_num=$(echo "$existing_url" | grep -oE '[0-9]+$')
  gh issue edit "$issue_num" --repo "$REPO" --body-file /tmp/bug-body.md
  echo "Updated issue #${issue_num}"
else
  # CREATE new issue
  issue_num=$(gh issue create \
    --repo "$REPO" \
    --title "$bug_title" \
    --body-file /tmp/bug-body.md \
    --label "$labels" \
    --json number -q .number)
  echo "Created issue #${issue_num}"
fi

repo=$(gh repo view --json nameWithOwner -q .nameWithOwner)
bug_url="https://github.com/$repo/issues/$issue_num"
```

### 5. Add to GitHub Project and Set Kanban Column

```bash
# Add issue to project
item_id=$(gh project item-add $PROJECT_NUMBER \
  --owner "$PROJECT_OWNER" \
  --url "$bug_url" \
  --format json 2>/dev/null | jq -r '.id' 2>/dev/null)

if [ -z "$item_id" ] || [ "$item_id" = "null" ]; then
  # Item might already exist, get its ID
  item_id=$(gh project item-list $PROJECT_NUMBER \
    --owner "$PROJECT_OWNER" \
    --format json 2>/dev/null | \
    jq -r --arg url "$bug_url" '.items[] | select(.content.url == $url) | .id' 2>/dev/null)
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

### 6. Update Source File with GitHub URL

```bash
current_date=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

if [ "$BUG_SOURCE" = "observation" ]; then
  # Update observation file
  if grep -q '^github:' "$BUG_PATH"; then
    sed -i.bak "/^github:/c\github: $bug_url" "$BUG_PATH"
  else
    sed -i.bak "/^---$/a\github: $bug_url" "$BUG_PATH"
  fi
  rm "${BUG_PATH}.bak" 2>/dev/null
else
  # Update progress.md or create github-mapping.md for session
  if [ -f "$BUG_PATH/progress.md" ]; then
    if grep -q '^github:' "$BUG_PATH/progress.md"; then
      sed -i.bak "/^github:/c\github: $bug_url" "$BUG_PATH/progress.md"
    else
      sed -i.bak "/^---$/a\github: $bug_url" "$BUG_PATH/progress.md"
    fi
    rm "$BUG_PATH/progress.md.bak" 2>/dev/null
  fi

  # Always create/update mapping file
  cat > "$BUG_PATH/github-mapping.md" << EOF
# GitHub Issue Mapping

Bug: #${issue_num} - ${bug_url}
Project: https://github.com/users/${PROJECT_OWNER}/projects/${PROJECT_NUMBER}
Kanban Status: ${KANBAN_NAME}

Synced: ${current_date}
EOF
fi
```

### 7. Output

```
✅ Bug synced to GitHub
  - Issue: #${issue_num}
  - Title: ${bug_title}
  - Status: ${bug_status:-unknown}
  - Kanban: ${KANBAN_NAME}
  - URL: ${bug_url}

Source: ${BUG_PATH}
```

## Kanban Logic Summary

| Bug State | Kanban Column |
|-----------|---------------|
| Observation (new bug report) | ToDo |
| Session in progress | In Progress |
| Session complete (all-fixed) | In Review |
| Session partial | In Progress |

## Error Handling

- If no bugs found, suggest running `/pm:bug-observe` or `/pm:bugfix`
- If GitHub issue creation fails, report error and don't update local files
- If project add fails, issue is still created - add to project manually
