---
allowed-tools: Bash, Read, Glob, AskUserQuestion
---

# PM Help

Interactive help for the PM workflow system.

## Instructions

Present the user with options using AskUserQuestion:

```yaml
AskUserQuestion:
  questions:
    - question: "What would you like to see?"
      header: "PM Help"
      options:
        - label: "Workflow Overview"
          description: "Commands and their sequence for features and bugs"
        - label: "Where Am I?"
          description: "Diagnose current state, find missing steps"
        - label: "Next Steps"
          description: "Show available tasks ready to start"
        - label: "Project Status"
          description: "PRD, epic, and task counts"
      multiSelect: false
```

Based on selection, show the appropriate content:

### If "Workflow Overview"

Display:

```
═══════════════════════════════════════════════════════════════
  🚀 GETTING STARTED
═══════════════════════════════════════════════════════════════

  /pm:init                   Initialize project (GitHub, board, workflow/)

═══════════════════════════════════════════════════════════════
  🎯 FEATURE WORKFLOW (Full)
═══════════════════════════════════════════════════════════════

  /pm:prd-new <name>         Design feature (Socratic questioning)
  /pm:epic-create <name>     Create implementation epic
  /pm:epic-decompose <name>  Break into tasks
  /pm:epic-sync <name>       Push to GitHub
  /pm:epic-start <name>      Create worktree + run tasks
  /pm:epic-verify <name>     Run tests + E2E
  /pm:epic-close <name>      Merge + cleanup

═══════════════════════════════════════════════════════════════
  ⚡ FEATURE WORKFLOW (One-Shot)
═══════════════════════════════════════════════════════════════

  /pm:epic-oneshot <name>    Create epic + worktree (no decomposition)
  /pm:epic-verify <name>     Run tests + E2E
  /pm:epic-close <name>      Merge + cleanup

  💡 Use for small, well-defined work (1-3 files)

═══════════════════════════════════════════════════════════════
  🐛 BUG WORKFLOW
═══════════════════════════════════════════════════════════════

  /pm:bugfix <observation>   All-in-one fix workflow

  Examples:
    /pm:bugfix "Video blank"     Asks questions first
    /pm:bugfix bugs.md           From file
    /pm:bugfix "Bug 1; Bug 2"   Multiple bugs

═══════════════════════════════════════════════════════════════
  🧪 TESTING
═══════════════════════════════════════════════════════════════

  Testing command hierarchy (which to use when):

  /pm:epic-verify <epic>       Orchestrator - REQUIRED before epic-close
    ↳ Runs unit tests
    ↳ Calls /testing:acceptance to generate E2E tests
    ↳ Calls /testing:e2e to run E2E tests
    ↳ Sets verified: true in epic frontmatter

  /testing:acceptance <epic>   Generate Playwright tests from acceptance criteria
  /testing:e2e [pattern]       Run existing Playwright tests

  💡 For epics: Use /pm:epic-verify (handles everything)
  💡 For standalone tests: Use /testing:e2e

═══════════════════════════════════════════════════════════════

  💡 Skip prd-new if feature is already well-defined
  💡 Use epic-oneshot for small changes (skips decompose/sync/start)
  💡 Run /pm:init first to setup project.yaml (once per project)
  💡 epic-verify is REQUIRED before epic-close (verification gate)
```

### If "Where Am I?"

Run diagnosis:

```bash
# Check in-progress epics
for epic_dir in workflow/epics/*/; do
  [ -d "$epic_dir" ] || continue
  epic_name=$(basename "$epic_dir")
  epic_file="$epic_dir/epic.md"
  [ -f "$epic_file" ] || continue

  status=$(grep "^status:" "$epic_file" 2>/dev/null | head -1 | sed 's/^status: *//')
  github=$(grep "^github:" "$epic_file" 2>/dev/null | head -1 | sed 's/^github: *//')
  oneshot=$(grep "^oneshot:" "$epic_file" 2>/dev/null | head -1 | sed 's/^oneshot: *//')
  task_count=$(ls "$epic_dir"/[0-9]*.md 2>/dev/null | wc -l)

  if [ "$status" = "in-progress" ]; then
    if [ "$oneshot" = "true" ]; then
      echo "📌 One-shot epic in progress: $epic_name"
    else
      echo "📌 Epic in progress: $epic_name"
      [ "$task_count" -eq 0 ] && echo "   ⚠️  No tasks. Run: /pm:epic-decompose $epic_name"
    fi
    [ -z "$github" ] || [ "$github" = "none" ] && echo "   ⚠️  Not synced. Run: /pm:epic-sync $epic_name"
  fi
done

# Check PRDs without epics
for prd in workflow/prds/*.md; do
  [ -f "$prd" ] || continue
  prd_name=$(basename "$prd" .md)
  [ ! -d "workflow/epics/$prd_name" ] && echo "📄 PRD without epic: $prd_name → Run: /pm:epic-create $prd_name"
done

# Check bug observations pending review
for obs in workflow/bugfixes/observations/*.md; do
  [ -f "$obs" ] || continue
  obs_name=$(basename "$obs" .md)
  status=$(grep "^status:" "$obs" 2>/dev/null | head -1 | sed 's/^status: *//')
  [ "$status" = "observed" ] && echo "🐛 Bug pending review: $obs_name → Run: /pm:bugfix $obs"
done

# Check in-progress bugfix sessions
for session_dir in workflow/bugfixes/20*/; do
  [ -d "$session_dir" ] || continue
  session_id=$(basename "$session_dir")
  progress_file="$session_dir/progress.md"
  [ -f "$progress_file" ] || continue

  phase=$(grep "^phase:" "$progress_file" 2>/dev/null | head -1 | sed 's/^phase: *//')
  if [ -n "$phase" ] && [ "$phase" -lt 8 ]; then
    echo "🔧 Bugfix in progress: $session_id (phase $phase)"
    echo "   ⚠️  Interrupted session. Resume in worktree: ../bugfix-$session_id"
  fi
done
```

Report findings with warnings highlighted. If no issues found, say:
```
✅ No workflow issues detected.
   Start with /pm:prd-new or /pm:epic-create or /pm:bugfix
```

### If "Next Steps"

```bash
found=0
for epic_dir in workflow/epics/*/; do
  [ -d "$epic_dir" ] || continue
  epic_name=$(basename "$epic_dir")

  for task_file in "$epic_dir"/[0-9]*.md; do
    [ -f "$task_file" ] || continue

    status=$(grep "^status:" "$task_file" | head -1 | sed 's/^status: *//')
    [ "$status" != "open" ] && [ -n "$status" ] && continue

    deps=$(grep "^depends_on:" "$task_file" | head -1 | sed 's/^depends_on: *//' | sed 's/^\[//' | sed 's/\]$//')

    if [ -z "$deps" ]; then
      task_name=$(grep "^name:" "$task_file" | head -1 | sed 's/^name: *//')
      task_num=$(basename "$task_file" .md)
      echo "✅ Ready: #$task_num - $task_name (epic: $epic_name)"
      ((found++))
    fi
  done
done

[ $found -eq 0 ] && echo "No tasks ready. Check /pm:blocked or create new epic"
echo ""
echo "📊 $found task(s) ready to start"
```

### If "Project Status"

```bash
# PRDs
prd_total=$(ls workflow/prds/*.md 2>/dev/null | wc -l)

# Epics
epic_total=$(ls -d workflow/epics/*/ 2>/dev/null | wc -l)
epic_backlog=$(grep -l "^status: *backlog" workflow/epics/*/epic.md 2>/dev/null | wc -l)
epic_progress=$(grep -l "^status: *in-progress" workflow/epics/*/epic.md 2>/dev/null | wc -l)
epic_complete=$(grep -l "^status: *completed" workflow/epics/*/epic.md 2>/dev/null | wc -l)

# Tasks
task_total=$(find workflow/epics -name "[0-9]*.md" 2>/dev/null | wc -l)
task_open=$(grep -l "^status: *open" workflow/epics/*/[0-9]*.md 2>/dev/null | wc -l)
task_closed=$(grep -l "^status: *closed" workflow/epics/*/[0-9]*.md 2>/dev/null | wc -l)

# Bugs
bug_observations=$(ls workflow/bugfixes/observations/*.md 2>/dev/null | wc -l)
bug_pending=$(grep -l "^status: *observed" workflow/bugfixes/observations/*.md 2>/dev/null | wc -l)
bug_sessions=$(ls -d workflow/bugfixes/20*/ 2>/dev/null | wc -l)
bug_in_progress=$(find workflow/bugfixes/20*/ -name "progress.md" -exec grep -l "^phase: [1-7]" {} \; 2>/dev/null | wc -l)
bug_complete=$(find workflow/bugfixes/20*/ -name "fix-report.md" 2>/dev/null | wc -l)

echo "📊 Project Status"
echo ""
echo "📄 PRDs: $prd_total"
echo "📚 Epics: $epic_total (Backlog: $epic_backlog | In Progress: $epic_progress | Complete: $epic_complete)"
echo "📝 Tasks: $task_total (Open: $task_open | Closed: $task_closed)"
echo "🐛 Bugs: $bug_observations observations (Pending: $bug_pending) | $bug_sessions sessions (In Progress: $bug_in_progress | Complete: $bug_complete)"
```
