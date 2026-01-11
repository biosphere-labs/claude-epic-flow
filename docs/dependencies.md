# Dependencies

Tools and MCP servers used by the workflow.

## Required

| Tool | Purpose | Install |
|------|---------|---------|
| [GitHub CLI](https://cli.github.com/) | Issue/PR automation, project boards | `brew install gh` then `gh auth login` |
| [Episodic Memory MCP](https://github.com/erichung9060/episodic-memory-mcp) | Conversation memory across sessions | `claude mcp add episodic-memory` |
| [Context7 MCP](https://github.com/upstash/context7) | Framework docs for best practices research | `claude mcp add context7` |

## Recommended

| Tool | Purpose | Install |
|------|---------|---------|
| [ast-grep](https://ast-grep.github.io/) | Structural code search (25+ languages) | `cargo install ast-grep --locked` |
| [Playwright](https://playwright.dev/) | E2E testing framework | `npm install -D playwright && npx playwright install` |

## MCP Server Configuration

After adding MCP servers, verify they're configured:

```bash
claude mcp list
```

## Minimum Setup

```bash
# GitHub CLI (required for all GitHub operations)
brew install gh && gh auth login

# Episodic memory (required for cross-session context)
claude mcp add episodic-memory

# ast-grep (recommended for code search)
cargo install ast-grep --locked
```
