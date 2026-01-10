# Claude Code for Solo Founders

A comprehensive project management and workflow system for Claude Code, designed for solo founders and small teams.

## Overview

This repository contains custom commands, rules, and agents that extend Claude Code with:

- **Project Management (PM) Workflow** - End-to-end feature development from PRD to deployment
- **Bug Tracking** - Observe, analyze, fix, and sync bugs to GitHub
- **Testing Integration** - E2E testing with Playwright, acceptance test generation
- **GitHub Sync** - Automatic sync to GitHub Issues and Project boards

## Directory Structure

```
.claude/
├── commands/           # Custom slash commands
│   ├── pm/            # Project management commands
│   ├── testing/       # Testing commands
│   └── context/       # Context management
├── rules/             # Behavioral rules for Claude
├── agents/            # Specialized agent definitions
├── hooks/             # Git and lifecycle hooks
├── scripts/           # Helper scripts
└── plugins/           # Plugin configurations
```

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
```

## Key Features

### Automatic GitHub Sync

All state changes automatically sync to GitHub:
- Epic creation → GitHub Issue created
- Task completion → Checklist updated
- Bug reports → Issue + Kanban board placement

### Verification Gate

`/pm:epic-close` requires verification:
- Must run `/pm:epic-verify` first
- Sets `verified: true` in epic frontmatter
- Ensures all tests pass before merge

### Testing Hierarchy

| Command | Purpose |
|---------|---------|
| `/pm:epic-verify` | Orchestrator - runs all tests |
| `/testing:acceptance` | Generate E2E tests from criteria |
| `/testing:e2e` | Run Playwright tests |

### Worktree-Based Development

Epics run in isolated git worktrees:
- No pollution of main checkout
- Parallel development possible
- Automatic dependency copying

## Project Configuration

Each project needs `.claude/project.yaml`:

```yaml
github:
  repo: "owner/repo"

project_board:
  project_number: 1
  project_owner: "owner"
  # ... status field IDs
```

Run `/pm:init` to auto-configure.

## Rules

Rules in `rules/` define Claude's behavior:
- `workflow-sequence.md` - Valid command sequences
- `worktree-operations.md` - Git worktree patterns
- `testing-philosophy.md` - Integration over mocks
- `agent-coordination.md` - Parallel agent work

## Installation

1. Clone to `~/.claude/`:
   ```bash
   git clone git@github.com:biosphere-labs/claude-code-for-solo-founder.git ~/.claude
   ```

2. Initialize your project:
   ```bash
   cd your-project
   claude
   > /pm:init
   ```

## Acknowledgments & Influences

| Project | Details |
|---------|---------|
| [CCPM](https://github.com/automazeio/ccpm) | Claude Code Project Manager - the original workflow system this evolved from |
| [Compounding Engineering](https://github.com/EveryInc/every-marketplace/tree/main/plugins/compounding-engineering) | Agent patterns from the Every plugin marketplace |
| [Episodic Memory MCP](https://github.com/erichung9060/episodic-memory-mcp) | Conversation memory and cross-session search |

## License

MIT
