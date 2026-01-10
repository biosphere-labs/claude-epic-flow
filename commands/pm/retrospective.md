---
allowed-tools: Bash, Read, Write, Task, mcp__episodic-memory__search, mcp__episodic-memory__read
---

# Retrospective: Lessons Learned

Analyze past sessions to extract mistakes, patterns, and lessons learned. Updates home CLAUDE.md with accumulated wisdom.

## Usage
```
/pm:retrospective [timeframe]
```

Where timeframe is optional:
- `today` - Today's sessions only
- `week` - Last 7 days (default)
- `month` - Last 30 days
- `all` - All available history

## Instructions

### 1. Search Episodic Memory for Problems

Use episodic memory to find conversations containing issues:

```
Search queries (run separately):
- "mistake error wrong fix"
- "redo retry again"
- "should have instead"
- "problem issue bug"
- "user frustrated confused"
- "misunderstood requirement"
```

### 2. Analyze Patterns

For each relevant conversation found, identify:

1. **What went wrong?**
   - Misunderstood requirements
   - Wrong approach chosen
   - Missed existing patterns
   - Over-engineered solution
   - Broke existing functionality

2. **Root cause?**
   - Didn't read enough context first
   - Made assumptions without asking
   - Ignored existing code patterns
   - Rushed without planning
   - Didn't test before committing

3. **What would have prevented it?**
   - Specific check to add
   - Question to ask first
   - Pattern to follow
   - Tool to use

### 3. Compile Findings

Create a structured summary:

```markdown
## Retrospective: {date}

### Mistakes Found
| Session | What Happened | Root Cause | Prevention |
|---------|---------------|------------|------------|
| {date} | {description} | {cause} | {rule} |

### Patterns Identified
- Pattern 1: {description}
- Pattern 2: {description}

### New Rules to Add
1. **Never Do**: {anti-pattern}
   - Why: {explanation}

2. **Always Do**: {best practice}
   - Why: {explanation}

### Existing Rules Reinforced
- {rule}: Violated {n} times
```

### 4. Update Home CLAUDE.md

Read current `~/.claude/CLAUDE.md` and append new learnings:

```markdown
## Lessons Learned (Auto-Updated)

Last updated: {date}

### Never Do
- ❌ {mistake 1} - learned {date}
- ❌ {mistake 2} - learned {date}

### Always Do
- ✅ {practice 1} - learned {date}
- ✅ {practice 2} - learned {date}

### Common Mistakes
| Mistake | Times | Last Seen |
|---------|-------|-----------|
| {description} | {count} | {date} |
```

### 5. Output Summary

```
✅ Retrospective complete

Sessions analyzed: {count}
Mistakes found: {count}
New rules added: {count}
Rules reinforced: {count}

Updated: ~/.claude/CLAUDE.md

Key learnings this period:
1. {learning 1}
2. {learning 2}
3. {learning 3}
```

## Automation

To run daily via cron:

```bash
# Add to crontab -e
0 9 * * * claude -p ~/.claude --dangerously-skip-permissions -c "/pm:retrospective today" >> ~/.claude/retrospective.log 2>&1
```

## Notes

- Uses episodic memory MCP for cross-session search
- Appends to CLAUDE.md, never overwrites
- Focuses on actionable rules, not blame
- Run weekly for best results
