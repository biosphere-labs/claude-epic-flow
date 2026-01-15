# Session Rename on Compaction

When a session is compacted (context summarization occurs), automatically rename the session to reflect its current work.

## When This Applies

This rule triggers whenever:
- The conversation context is compacted/summarized
- You receive a system message indicating summarization occurred
- The context window is being managed through summarization

## Action Required

Immediately after compaction:

1. **Rename the session** using the `/rename` command:
   ```
   /rename <descriptive-name>
   ```

2. **Update the terminal title** for Tilix using the escape sequence:
   ```bash
   echo -ne "\033]0;<session-name> | $(echo $PWD | rev | cut -d'/' -f1-3 | rev)\007"
   ```
   This sets the Tilix tab/terminal title to show both the session name and the last 3 path segments (e.g., `dev/workspaces/my-project`), making it easy to distinguish between multiple Claude Code terminals.

## Naming Guidelines

Derive the session name from:
1. The main topic or task from the compaction summary
2. Keep it concise (2-4 words, use hyphens)
3. Make it searchable and meaningful

**Good names:**
- `auth-flow-refactor`
- `fix-database-timeout`
- `epic-user-dashboard`
- `debug-api-errors`
- `claude-code-cwd-research`

**Avoid:**
- Generic names like `session-1` or `work`
- Overly long descriptions
- Timestamps (the system tracks those)

## Why This Matters

- Sessions terminated unexpectedly will still have meaningful names
- Makes `/resume` and `claude --resume` useful for finding past work
- Protects against path changes orphaning session history
- Creates a searchable archive of past work
- **Tilix terminal titles** help distinguish between multiple Claude Code sessions running in parallel

## Example

After seeing a compaction summary like:
> "Discussed implementing user authentication with OAuth2, reviewed existing auth patterns..."

Rename and update terminal title:
```
/rename oauth2-auth-implementation
```
```bash
echo -ne "\033]0;oauth2-auth-implementation | $(echo $PWD | rev | cut -d'/' -f1-3 | rev)\007"
```

This will show something like `oauth2-auth-implementation | dev/workspaces/my-project` in the Tilix tab.

## Frequency

Rename after each compaction. If the focus shifts significantly during a long session, the new name after the next compaction will reflect the current work - this is fine and expected.
