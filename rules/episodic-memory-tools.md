# Episodic Memory Tool Usage

When working with conversation history files (`.jsonl` files in `~/.claude/projects/` or `~/.config/superpowers/conversation-archive/`):

## Use the Correct Tools

**DO use `mcp__episodic-memory__search`** for:
- Finding past conversations about a topic
- Searching for decisions, solutions, or discussions
- Semantic similarity search across conversation history

**DO use `mcp__episodic-memory__read`** for:
- Reading full conversation content from search results
- Use the `startLine` and `endLine` parameters from search results
- Example: If search shows "Lines 950-1000 in /path/to/file.jsonl", call read with those line numbers

**DO NOT use the native `Read` tool** for:
- Conversation JSONL files - they exceed the 25000 token limit
- Files in `~/.claude/projects/` or `conversation-archive/` directories

## Why This Matters

- Native `Read` tool has a 25000 token limit
- Conversation files can be 30MB+ with 50,000+ tokens
- Episodic memory's `read` tool has no limit and formats output properly
- Using the wrong tool causes `MaxFileReadTokenExceededError`

## Correct Pattern

```
1. Search: mcp__episodic-memory__search with query
2. Review: Look at snippets and line ranges in results
3. Read: mcp__episodic-memory__read with path AND startLine/endLine from search
```
