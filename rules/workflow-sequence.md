# Workflow Sequence Rule

This rule defines the valid sequences through the PM workflow and prerequisites for each command. Commands should validate prerequisites and warn users about skipped steps.

## Valid Workflow Paths

### Project Setup (Once per project)
```
init
```
Run once to configure `.claude/project.yaml` with GitHub repo and project board settings.

### Feature Workflow (Full)
```
init → prd-new → epic-create → epic-decompose → epic-sync → epic-start → epic-verify → epic-close
```

### Feature Workflow (Quick)
```
init → epic-create → epic-decompose → epic-sync → epic-start → epic-verify → epic-close
```
Use when feature is already well-defined (no PRD needed).

### Bug Workflow
```
init → bugfix (all-in-one)
```
Or expanded:
```
init → bug-observe → bugfix → bug-sync
```

## Command Prerequisites

### `/pm:init`
**Prerequisites:** Git repository (recommended: with GitHub remote)
**Creates:** `.claude/project.yaml`, `workflow/` directory structure
**Next:** Any other PM command

### `/pm:prd-new <name>`
**Prerequisites:** Project initialized (`.claude/project.yaml` exists)
**Creates:** `workflow/prds/<name>.md`
**Next:** `/pm:epic-create <name>`

### `/pm:epic-create <name>`
**Prerequisites:**
- Optional: PRD exists at `workflow/prds/<name>.md`
- If no PRD, user should confirm they want to skip PRD phase

**Preflight check:**
```bash
if [ ! -f "workflow/prds/$NAME.md" ]; then
  warn "No PRD found. Creating epic without design document."
  # Prompt: Continue without PRD? or Run /pm:prd-new first?
fi
```

**Creates:** `workflow/epics/<name>/epic.md`
**Next:** `/pm:epic-decompose <name>`

### `/pm:epic-decompose <name>`
**Prerequisites:**
- Epic must exist: `workflow/epics/<name>/epic.md`
- Epic should not already have tasks (or confirm overwrite)

**Preflight check:**
```bash
if [ ! -f "workflow/epics/$NAME/epic.md" ]; then
  error "Epic not found. Run: /pm:epic-create $NAME"
  exit 1
fi

if ls workflow/epics/$NAME/[0-9]*.md 2>/dev/null | head -1 > /dev/null; then
  warn "Tasks already exist. This will add more tasks."
  # Prompt: Continue? or View existing tasks first?
fi
```

**Creates:** `workflow/epics/<name>/001.md`, `002.md`, etc.
**Next:** `/pm:epic-sync <name>`

### `/pm:epic-sync <name>`
**Prerequisites:**
- Epic must exist with tasks
- GitHub CLI authenticated

**Preflight check:**
```bash
if [ ! -f "workflow/epics/$NAME/epic.md" ]; then
  error "Epic not found. Run: /pm:epic-create $NAME"
  exit 1
fi

task_count=$(ls workflow/epics/$NAME/[0-9]*.md 2>/dev/null | wc -l)
if [ "$task_count" -eq 0 ]; then
  error "No tasks found. Run: /pm:epic-decompose $NAME"
  exit 1
fi
```

**Creates:** GitHub issue, updates `github:` field in epic.md
**Next:** `/pm:epic-start <name>`

### `/pm:epic-start <name>`
**Prerequisites:**
- Epic must exist with tasks
- Should be synced to GitHub (warn if not)

**Preflight check:**
```bash
if [ ! -f "workflow/epics/$NAME/epic.md" ]; then
  error "Epic not found. Run: /pm:epic-create $NAME"
  exit 1
fi

task_count=$(ls workflow/epics/$NAME/[0-9]*.md 2>/dev/null | wc -l)
if [ "$task_count" -eq 0 ]; then
  error "No tasks found. Run: /pm:epic-decompose $NAME"
  exit 1
fi

github=$(grep "^github:" workflow/epics/$NAME/epic.md | sed 's/^github: *//')
if [ -z "$github" ] || [ "$github" = "none" ]; then
  warn "Epic not synced to GitHub. Progress won't be tracked online."
  # Prompt: Continue anyway? or Run /pm:epic-sync first?
fi
```

**Creates:** Worktree, starts dev servers
**Next:** `/pm:epic-verify <name>`

### `/pm:epic-verify <name>`
**Prerequisites:**
- Epic must exist
- Should have some closed tasks (warn if all open)

**Preflight check:**
```bash
if [ ! -f "workflow/epics/$NAME/epic.md" ]; then
  error "Epic not found"
  exit 1
fi

closed=$(grep -l "^status: *closed" workflow/epics/$NAME/[0-9]*.md 2>/dev/null | wc -l)
if [ "$closed" -eq 0 ]; then
  warn "No tasks completed yet. Running tests on incomplete work."
fi
```

**Creates:** Test results, potentially more tests
**Next:** `/pm:epic-close <name>`

### `/pm:epic-close <name>`
**Prerequisites:**
- Epic must be verified (`verified: true` in frontmatter)
- All tasks should be closed (warn if not)

**Preflight check:**
```bash
# REQUIRED: Check verification
verified=$(grep "^verified:" workflow/epics/$NAME/epic.md | sed 's/verified: *//')
if [ "$verified" != "true" ]; then
  error "Epic not verified. Run: /pm:epic-verify $NAME"
  exit 1
fi

# Check for open tasks
open=$(grep -l "^status: *open" workflow/epics/$NAME/[0-9]*.md 2>/dev/null | wc -l)
if [ "$open" -gt 0 ]; then
  warn "$open task(s) still open"
  # Prompt: Close anyway? or Complete tasks first?
fi
```

**Creates:** Merged PR, cleaned up worktree
**Next:** Done (or start new epic)

### `/pm:bugfix <observation>`
**Prerequisites:** None (entry point)
**Creates:** `workflow/bugfixes/<session>/` directory
**Next:** Done (PR created)

## Warning Levels

### Error (blocks execution)
- Required file/directory doesn't exist
- Critical prerequisite missing

### Warning (asks to continue)
- Recommended step skipped
- Potentially incomplete state
- Unusual sequence

## Implementing Preflight Checks

Commands should include preflight validation at the start. Use `gum confirm` for interactive prompts:

```bash
# In command script or markdown instructions
preflight_check() {
  # Check prerequisites
  if [ ! -f "workflow/epics/$NAME/epic.md" ]; then
    echo "❌ Epic not found: $NAME"
    echo "   Run: /pm:epic-create $NAME"
    exit 1
  fi

  # Check recommended steps
  github=$(grep "^github:" workflow/epics/$NAME/epic.md | sed 's/^github: *//')
  if [ -z "$github" ]; then
    if command -v gum &> /dev/null; then
      gum confirm "Epic not synced to GitHub. Continue anyway?" || exit 1
    else
      echo "⚠️  Epic not synced to GitHub"
      read -p "Continue anyway? (y/n) " -n 1 -r
      [[ ! $REPLY =~ ^[Yy]$ ]] && exit 1
    fi
  fi
}
```

## Sequence Diagram

```
                    ┌─────────────┐
                    │   Start     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    init     │◄─── Run once per project
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
      ┌───────────────┐         ┌───────────────┐
      │  prd-new      │         │  bugfix       │
      │  (optional)   │         │  (all-in-one) │
      └───────┬───────┘         └───────┬───────┘
              │                         │
              ▼                         ▼
      ┌───────────────┐         ┌───────────────┐
      │  epic-create  │         │     Done      │
      └───────┬───────┘         └───────────────┘
              │
              ▼
      ┌───────────────┐
      │ epic-decompose│
      └───────┬───────┘
              │
              ▼
      ┌───────────────┐
      │  epic-sync    │◄─── Can skip (local only)
      └───────┬───────┘
              │
              ▼
      ┌───────────────┐
      │  epic-start   │
      └───────┬───────┘
              │
              ▼
      ┌───────────────┐
      │  epic-verify  │
      └───────┬───────┘
              │
              ▼
      ┌───────────────┐
      │  epic-close   │
      └───────┬───────┘
              │
              ▼
      ┌───────────────┐
      │     Done      │
      └───────────────┘
```

## Quick Reference

| Command | Requires | Creates | Warn If Missing |
|---------|----------|---------|-----------------|
| init | git repo | project.yaml + workflow/ | - |
| prd-new | project.yaml | PRD file | - |
| epic-create | project.yaml | Epic dir + epic.md | PRD |
| epic-decompose | epic.md | Task files | - |
| epic-sync | epic.md + tasks | GitHub issue | - |
| epic-start | epic.md + tasks | Worktree | GitHub sync |
| epic-verify | epic.md | Test results + verified flag | Completed tasks |
| epic-close | epic.md + verified: true | Merged PR | Open tasks |
| bugfix | project.yaml | Bugfix session | - |
