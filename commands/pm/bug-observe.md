---
description: Transform a bug observation into a detailed bug report with context from episodic memory, git history, and documents
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Task, mcp__episodic-memory__search, mcp__episodic-memory__read, mcp__basic-memory__search_notes, mcp__basic-memory__build_context
---

# Bug Observation → Detailed Report

Transform a rough bug observation into a comprehensive bug report with context, ready for `/pm:bugfix`.

## Usage
```
/pm:bug-observe <observation>
```

**Parameter:**
- `$ARGUMENTS`: Required. The bug observation (what you noticed, symptoms, when it happens)

## Examples
```bash
# Simple observation
/pm:bug-observe "Login button doesn't respond on mobile"

# Detailed observation
/pm:bug-observe "When I click on a video in the library, sometimes it shows a blank page instead of the video content. Happens more often after navigating back from another page."
```

## Required Rules

**IMPORTANT:** Before executing, read:
- `.claude/rules/datetime.md` - For timestamps
- `.claude/rules/episodic-memory-tools.md` - For memory tool usage

## Phase 0: Prime Context (ALWAYS RUN FIRST)

Before any research or user interaction, load project context:

1. **Check context exists:**
   ```bash
   ls -la .claude/context/*.md 2>/dev/null | wc -l
   ```
   - If 0 files: Tell user "❌ No context found. Run /ccpmcontext:create first." and stop.

2. **Load context files in order:**
   - Read `.claude/context/project-overview.md` - Project understanding
   - Read `.claude/context/tech-context.md` - Technical stack
   - Read `.claude/context/progress.md` - Current status
   - Read `.claude/context/system-patterns.md` - Architecture patterns
   - Read `.claude/context/project-structure.md` - File organization

3. **Brief acknowledgment (don't be verbose):**
   ```
   🧠 Context loaded ({n} files). Project: {name}, Branch: {branch}
   ```

## Instructions

You are creating a detailed bug report from the observation: **$ARGUMENTS**

### Phase 1: Context Gathering (Parallel)

Launch these searches in parallel using Task tool:

**1. Episodic Memory Search:**
> Use `mcp__episodic-memory__search` with query derived from the observation
> Look for: past discussions about this feature, previous bugs, design decisions
> Extract: relevant context, what was tried before, known issues

**2. Git History Search:**
> Use Task tool with subagent_type "compound-engineering:research:git-history-analyzer"
> Prompt: "Analyze recent git history related to: {keywords from observation}. Find commits in the last 2 weeks that touched related files. Identify what changed recently that might have caused this. Return: list of relevant commits with their descriptions and what they changed."

**3. Document Search:**
> Use `mcp__basic-memory__search_notes` or Grep to find related documentation
> Search for: feature specs, design docs, known issues in docs/

**4. Codebase Search:**
> Use Task tool with subagent_type "Explore"
> Prompt: "Find code related to: {feature from observation}. Identify the files involved, the component structure, and any error handling or edge cases that might be relevant."

### Phase 2: Synthesize Context

After parallel searches complete:

1. **Correlate findings:**
   - Did episodic memory reveal past similar bugs?
   - Did recent commits touch related code?
   - Are there known limitations in docs?

2. **Identify likely root causes:**
   - Based on context, what are probable causes?
   - Rank by likelihood

3. **Determine acceptance criteria:**
   - What should work when the bug is fixed?
   - What shouldn't break?

### Phase 3: Create Bug Report

Create timestamped bug report file:

```bash
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p workflow/bugfixes/observations
```

Write to `workflow/bugfixes/observations/bug-$TIMESTAMP.md`:

```markdown
---
id: BUG-$TIMESTAMP
status: observed
created: {ISO timestamp}
priority: {inferred: critical|high|medium|low}
observation_source: user
---

# Bug: {synthesized title from observation}

## Original Observation

{$ARGUMENTS verbatim}

## Description

{Expanded, clear description based on observation and context}

## Reproduction Steps

1. {Step 1 - inferred from observation and code analysis}
2. {Step 2}
3. {Expected: ...}
4. {Actual: ...}

## Context from Memory

### Related Past Discussions
{Summary of episodic memory findings, with conversation references}

### Recent Relevant Changes
| Commit | Date | Description | Files |
|--------|------|-------------|-------|
| {hash} | {date} | {message} | {files} |

### Related Documentation
{Links to relevant docs found}

## Technical Analysis

### Likely Affected Files
- {file1}: {why relevant}
- {file2}: {why relevant}

### Probable Root Causes
1. **Most likely:** {cause} - {evidence}
2. **Alternative:** {cause} - {evidence}

### Related Code Patterns
{Any relevant code snippets or patterns found}

## Acceptance Criteria

When fixed, the following must be true:
- [ ] {criterion 1}
- [ ] {criterion 2}
- [ ] {criterion 3}

## Regression Risks

Areas that might break if not careful:
- {risk 1}
- {risk 2}

## Suggested Fix Approach

{Based on analysis, suggested approach to fix}

---

**Ready for:** `/pm:bugfix workflow/bugfixes/observations/bug-$TIMESTAMP.md`
```

### Phase 4: Output Summary

```
═══════════════════════════════════════════════════════════════
  ✅ BUG REPORT CREATED
═══════════════════════════════════════════════════════════════

  Observation: {first 50 chars of $ARGUMENTS}...
  Report: workflow/bugfixes/observations/bug-$TIMESTAMP.md
  Priority: {priority}

  Context gathered:
    - Episodic memory: {n} related discussions
    - Git history: {n} recent relevant commits
    - Documents: {n} related docs

  Probable root cause: {most likely cause}

  📍 Workflow position: Bug documented, ready to fix

  ➡️  Next steps:
     /pm:bugfix bug-$TIMESTAMP.md   ← Start fixing (recommended)
     /pm:bug-sync bug-$TIMESTAMP.md ← Sync to GitHub first

  🔄 Related:
     /pm:bugfix "description"       ← Quick fix without report
     /pm:status                     ← Overall workflow status

═══════════════════════════════════════════════════════════════
```

## Output Location

Bug reports are saved to:
- `workflow/bugfixes/observations/bug-{timestamp}.md`

After processing by `/pm:bugfix`, session files go to:
- `workflow/bugfixes/{session-id}/`
