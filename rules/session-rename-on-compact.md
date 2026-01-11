# Session Rename on Compaction

When a session is compacted (context summarization occurs), automatically rename the session to reflect its current work.

## When This Applies

This rule triggers whenever:
- The conversation context is compacted/summarized
- You receive a system message indicating summarization occurred
- The context window is being managed through summarization

## Action Required

Immediately after compaction, rename the session using the `/rename` command:

```
/rename <descriptive-name>
```

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

## Example

After seeing a compaction summary like:
> "Discussed implementing user authentication with OAuth2, reviewed existing auth patterns..."

Rename to:
```
/rename oauth2-auth-implementation
```

## Frequency

Rename after each compaction. If the focus shifts significantly during a long session, the new name after the next compaction will reflect the current work - this is fine and expected.
