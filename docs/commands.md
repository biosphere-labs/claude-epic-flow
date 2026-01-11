# Commands Reference

Complete reference for all slash commands.

## PM Workflow Commands

### Feature Development

```
/pm:init                    # Initialize project (once per project)
/pm:prd-new <name>          # Create PRD via Socratic questioning
/pm:epic-create <name>      # Create implementation epic
/pm:epic-decompose <name>   # Break into tasks
/pm:epic-start <name>       # Create worktree + start dev servers
/pm:epic-verify <name>      # Run tests (REQUIRED before close)
/pm:epic-close <name>       # Merge to staging + cleanup
```

### Bug Fixing

```
/pm:bugfix <observation>    # All-in-one bug fix workflow
/pm:bug-observe <desc>      # Create detailed bug report
```

### Status & Help

```
/pm:help                    # Interactive help
/pm:status                  # Current workflow status
/pm:in-progress             # Show active work
/pm:standup                 # Generate standup summary
/pm:retrospective           # Lessons learned from completed work
```

### Epic Management

```
/pm:epic-all                # List all epics
/pm:epic-edit <name>        # Edit epic details
/pm:epic-sync <name>        # Sync epic to GitHub
```

### PRD Management

```
/pm:prd-list                # List all PRDs
/pm:prd-sync <name>         # Sync PRD to GitHub
```

## Testing Commands

```
/testing:e2e                # Run Playwright E2E tests
/testing:acceptance         # Generate acceptance tests from criteria
```

## Context Commands

```
/context:create             # Create initial context for project
/context:update             # Update context with recent changes
/context:prime              # Load context for new session
```

## Workflow Sequence

Valid progression through commands:

```
init → prd-new → epic-create → epic-decompose → epic-sync → epic-start → epic-verify → epic-close
       (optional)
```

For bugs:
```
init → bugfix (all-in-one)
```

## Verification Gate

`/pm:epic-close` requires verification:
- Must run `/pm:epic-verify` first
- Sets `verified: true` in epic frontmatter
- Ensures all tests pass before merge
