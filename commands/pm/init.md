---
allowed-tools: Bash, Write, LS
---

# Initialize PM Workflow

Set up the workflow/ directory structure for a new project.

## Usage
```
/pm:init
```

## Instructions

### 1. Check if Already Initialized

```bash
if [ -d "workflow" ]; then
  echo "⚠️  workflow/ already exists"
  ls -la workflow/
fi
```

If exists, ask user if they want to continue (will add missing folders only).

### 2. Create Directory Structure

```bash
# Core workflow folders
mkdir -p workflow/prds/.archived
mkdir -p workflow/epics/.archived
mkdir -p workflow/bugfixes/.archived
mkdir -p workflow/bugfixes/observations
mkdir -p workflow/brainstorms/.archived
mkdir -p workflow/context

# Add .gitkeep to empty folders
for dir in workflow/prds/.archived workflow/epics/.archived workflow/bugfixes/.archived workflow/brainstorms/.archived; do
  touch "$dir/.gitkeep"
done
```

### 3. Create Context Placeholders

Create minimal context files if they don't exist:

**workflow/context/project-overview.md:**
```markdown
# Project Overview

## Purpose
[Describe what this project does]

## Key Features
- [Feature 1]
- [Feature 2]

## Tech Stack
- [Technology 1]
- [Technology 2]
```

**workflow/context/tech-context.md:**
```markdown
# Technical Context

## Architecture
[Describe the architecture]

## Key Patterns
[Describe patterns used]

## Dependencies
[List key dependencies]
```

### 4. Create .gitignore Entry

Add workflow patterns to .gitignore if not present:

```bash
if ! grep -q "workflow/\*\*/\.archived" .gitignore 2>/dev/null; then
  echo "" >> .gitignore
  echo "# Workflow archives (optional - uncomment to ignore)" >> .gitignore
  echo "# workflow/**/.archived/" >> .gitignore
fi
```

### 5. Output Summary

```
✅ PM workflow initialized

Created:
  workflow/
  ├── prds/          (product requirements)
  │   └── .archived/
  ├── epics/         (implementation epics)
  │   └── .archived/
  ├── bugfixes/      (bug tracking)
  │   ├── observations/
  │   └── .archived/
  ├── brainstorms/   (ideas and designs)
  │   └── .archived/
  └── context/       (project context docs)

Next steps:
  1. Run /context:create to generate context docs
  2. Run /pm:prd-new <name> to start your first feature
  3. Or /pm:help to see all available commands
```

## Notes

- This command is idempotent - safe to run multiple times
- Only creates missing directories/files
- Does not overwrite existing content
- Archive folders keep completed work out of the way
