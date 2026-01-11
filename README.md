# Claude Epic Flow

Epic-driven project management for Claude Code.

## What It Does

- **PRD → Epic → Tasks → Code → Tests → Merge** - complete feature workflow
- **Automatic GitHub sync** - issues, checklists, project boards
- **Worktree isolation** - parallel development without pollution
- **Verification gates** - tests must pass before merge

## Quick Start

```bash
# 1. Clone to ~/.claude/
git clone git@github.com:biosphere-labs/claude-epic-flow.git ~/.claude

# 2. Install minimum dependencies
brew install gh && gh auth login
claude mcp add episodic-memory

# 3. Initialize your project
cd your-project
claude
> /pm:init
```

## Core Commands

```
/pm:epic-create <name>      # Start a feature
/pm:epic-decompose <name>   # Break into tasks
/pm:epic-start <name>       # Create worktree, begin work
/pm:epic-verify <name>      # Run tests
/pm:epic-close <name>       # Merge + cleanup

/pm:bugfix <observation>    # All-in-one bug fix
/pm:help                    # Interactive help
```

## Documentation

- [Commands Reference](docs/commands.md) - all slash commands
- [Configuration](docs/configuration.md) - project setup and rules
- [Dependencies](docs/dependencies.md) - required and recommended tools
- [System Context](docs/system-context.md) - context for Claude agents

## Compatible Projects

- [Claude Code Scheduler](https://github.com/biosphere-labs/claude-code-scheduler)
- [Epic Executor](https://github.com/biosphere-labs/epic-executor)

## Acknowledgments

| Project | Details |
|---------|---------|
| [CCPM](https://github.com/automazeio/ccpm) | Original workflow system this evolved from |
| [Compounding Engineering](https://github.com/EveryInc/every-marketplace/tree/main/plugins/compounding-engineering) | Agent patterns |

## License

MIT
