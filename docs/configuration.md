# Project Configuration

Each project using this workflow needs a `.claude/project.yaml` file.

## Auto-Configuration

Run `/pm:init` in your project to auto-configure:

```bash
cd your-project
claude
> /pm:init
```

This creates the config and `workflow/` directory structure.

## Manual Configuration

Create `.claude/project.yaml`:

```yaml
project:
  name: "my-project"
  description: "Project description"

github:
  repo: "owner/repo"
  default_branch: "main"  # or "staging" for two-branch model

project_board:
  project_number: 1
  project_owner: "owner"
  # Optional: status field IDs for Kanban automation
```

## Directory Structure

After initialization:

```
your-project/
├── .claude/
│   └── project.yaml      # Project config
└── workflow/
    ├── prds/             # Product requirement docs
    ├── epics/            # Epic definitions and tasks
    ├── bugfixes/         # Bug fix sessions
    └── context/          # Project context files
```

## Two-Branch Model

For projects using staging:

```
feature/xyz ──→ staging ──→ main
              (PR)       (PR)
```

Set `default_branch: "staging"` in config.

## Rules

Rules in `~/.claude/rules/` define Claude's behavior across all projects:

| Rule | Purpose |
|------|---------|
| `workflow-sequence.md` | Valid command sequences |
| `worktree-operations.md` | Git worktree patterns |
| `testing-philosophy.md` | Integration over mocks |
| `agent-coordination.md` | Parallel agent work |
| `session-rename-on-compact.md` | Auto-rename on context compaction |
