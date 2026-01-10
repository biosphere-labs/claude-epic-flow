# Workflow Visual Diagram

Three entry points into the development workflow, each with different starting contexts.

## Branch Model

This project uses a **two-branch model**:

```
feature/xyz ──→ staging ──→ main
              (PR)       (PR)
```

- **staging**: Integration branch for testing (deploys to staging environment)
- **main**: Production branch (deploys to production environment)
- **feature/epic branches**: Created from staging, merged back to staging via PR

---

## Entry Point 1: Feature Development (From Idea)

**When to use:** New feature ideas that need research and design before implementation

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           FEATURE DEVELOPMENT FLOW                                  │
└─────────────────────────────────────────────────────────────────────────────────────┘

  USER INPUT                    COMMAND                        OUTPUT LOCATION
  ──────────────────────────────────────────────────────────────────────────────────

  "Your rough idea"        ──▶  /brainstorm                ──▶ workflow/brainstorms/brainstorm_*.md
  (text description)            │                               │
                                │ Researches competitors,       │ Design document with:
                                │ asks clarifying questions,    │ • Problem definition
                                │ explores UX patterns          │ • Proposed approach
                                │                               │ • Technical decisions
                                ▼                               ▼

  brainstorm file path     ──▶  /pm:plan-analyze           ──▶ .claude/plans/<name>.md
                                │                               │
                                │ Runs spec-flow-analyzer,      │ Full implementation plan:
                                │ searches episodic memory,     │ • User flow analysis
                                │ explores codebase             │ • Gap identification
                                │                               │ • Architecture breakdown
                                │                               │ • Acceptance criteria
                                ▼                               ▼

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │  DECISION POINT: Is a formal PRD needed?                                        │
  │                                                                                  │
  │  YES (larger features)        │           NO (smaller features)                 │
  │  ─────────────────────────────┼─────────────────────────────────────────────    │
  │                               │                                                 │
  │  feature name           ──▶  /pm:prd-new                  feature name ─────┐   │
  │                               │                                              │   │
  │                               │ Interactive brainstorm,                      │   │
  │                               │ asks clarifying questions                    │   │
  │                               ▼                                              │   │
  │                         workflow/prds/<name>.md                               │   │
  │                               │                                              │   │
  │                               │ PRD with user stories,                       │   │
  │                               │ requirements, success criteria               │   │
  │                               ▼                                              │   │
  │                                                                              │   │
  │  feature name           ──▶  /pm:epic-create                                 │   │
  │                               │                                              │   │
  │                               │ Converts PRD to technical epic               │   │
  │                               ▼                                              │   │
  │                         workflow/epics/<name>/epic.md ◀───────────────────────┘   │
  │                                                                                  │
  └──────────────────────────────────┬───────────────────────────────────────────────┘
                                     │
                                     ▼

  feature name             ──▶  /pm:epic-decompose         ──▶ workflow/epics/<name>/001.md
                                │                               workflow/epics/<name>/002.md
                                │ Breaks epic into tasks,       workflow/epics/<name>/...
                                │ sets dependencies,            │
                                │ estimates effort              │ Task files with:
                                │                               │ • Acceptance criteria
                                ▼                               │ • depends_on fields
                                                                │ • parallel flags
                                                                ▼

  feature name             ──▶  /pm:epic-start    ──▶ .worktrees/epic/<name>/
                                │                               │
                                │ Creates git worktree,         │ Isolated development
                                │ launches parallel agents,     │ environment with:
                                │ begins implementation         │ • Feature branch
                                │                               │ • Task commits
                                ▼                               ▼

  feature name             ──▶  /pm:epic-verify               ──▶ Verification report
                                │
                                │ Unit tests, writes missing
                                │ tests, E2E via Playwright
                                ▼

  feature name             ──▶  /pm:epic-close             ──▶ Merged to staging
                                │
                                │ Syncs GitHub, merges to
                                │ staging, cleans worktree
                                ▼

                               staging ──▶ PR to main ──▶ ✅ SHIPPED
```

---

## Entry Point 2: Bug Fixing (From Observation)

**When to use:** Known bugs that need investigation and fixing

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              BUG FIXING FLOW                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘

  USER INPUT                    COMMAND                        OUTPUT LOCATION
  ──────────────────────────────────────────────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │  DECISION POINT: Is context gathering needed?                                   │
  │                                                                                 │
  │  YES (complex/unfamiliar bug)     │       NO (simple/obvious bug)              │
  │  ─────────────────────────────────┼──────────────────────────────────────────   │
  │                                   │                                             │
  │  "Bug observation text"     ──▶  /pm:bug-observe                               │
  │  (symptoms, when it happens)      │                                             │
  │                                   │ Searches episodic memory,                   │
  │                                   │ git history, documents,                     │
  │                                   │ analyzes codebase                           │
  │                                   ▼                                             │
  │                             workflow/bugfixes/observations/bug-<timestamp>.md   │
  │                                   │                                             │
  │                                   │ Detailed bug report with:                   │
  │                                   │ • Root cause hypothesis                     │
  │                                   │ • Related past discussions                  │
  │                                   │ • Recent relevant commits                   │
  │                                   │ • Acceptance criteria                       │
  │                                   ▼                                             │
  │                                   │                                             │
  └───────────────────────────────────┼─────────────────────────────────────────────┘
                                      │
                                      ▼

  bug report path          ──▶  /pm:bugfix                 ──▶ workflow/bugfixes/<session>/
  OR inline description         │                               │
  OR bug list file              │ Creates worktree,             │ Session files:
                                │ analyzes in parallel,         │ • BUG-001.md
                                │ fixes bugs,                   │ • BUG-002.md
                                │ runs E2E test loop            │ • fix-report.md
                                ▼                               ▼

                              ✅ BUGS FIXED (with commits)
```

---

## Entry Point 3: Plan Import (From Existing Document)

**When to use:** You have an existing plan, spec, or design document that needs to become a formal PRD

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           PLAN IMPORT FLOW                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘

  USER INPUT                    COMMAND                        OUTPUT LOCATION
  ──────────────────────────────────────────────────────────────────────────────────

  existing plan/spec file  ──▶  /pm:plan-analyze           ──▶ .claude/plans/<name>.md
  (any markdown document)       │                               │
                                │ Runs spec-flow-analyzer,      │ Analyzed plan with:
                                │ identifies gaps,              │ • User flows mapped
                                │ researches context            │ • Gaps identified
                                │                               │ • Acceptance criteria
                                ▼                               ▼

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │  DECISION POINT: Need formal PRD or go direct to epic?                          │
  │                                                                                 │
  │  Create PRD (formal tracking)     │     Direct to Epic (faster)                │
  │  ─────────────────────────────────┼──────────────────────────────────────────   │
  │                                   │                                             │
  │  /pm:prd-new <name>               │     /pm:epic-decompose <name>               │
  │  (use plan as context)            │     (directly creates tasks from plan)      │
  │         │                         │             │                               │
  │         ▼                         │             │                               │
  │  workflow/prds/<name>.md           │             │                               │
  │         │                         │             │                               │
  │         ▼                         │             │                               │
  │  /pm:epic-create <name>           │             │                               │
  │         │                         │             ▼                               │
  │         └─────────────────────────┴──▶ workflow/epics/<name>/                   │
  │                                                                                 │
  └─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼

                          (continues with decompose → start → verify → close)
```

---

## Quick Command Reference

| Command | Input Required | Output Location |
|---------|----------------|-----------------|
| `/brainstorm` | Text description of idea | `workflow/brainstorms/brainstorm_*.md` |
| `/pm:plan-analyze` | Path to brainstorm/plan file | `.claude/plans/<name>.md` |
| `/pm:prd-new` | Feature name | `workflow/prds/<name>.md` |
| `/pm:epic-create` | Feature name (requires PRD) | `workflow/epics/<name>/epic.md` |
| `/pm:epic-decompose` | Feature name (requires epic) | `workflow/epics/<name>/001.md, 002.md, ...` |
| `/pm:epic-start` | Feature name | `.worktrees/epic/<name>/` (git worktree) |
| `/pm:epic-verify` | Feature name | Verification report (unit tests + E2E) |
| `/pm:epic-close` | Feature name | Merged to staging |
| `/pm:bug-observe` | Bug observation text | `workflow/bugfixes/observations/bug-*.md` |
| `/pm:bugfix` | Bug report path or inline text | `workflow/bugfixes/<session>/` |

---

## Decision Tree Summary

```
START: What do you have?
        │
        ├── A rough idea ──────────────▶ /brainstorm → /pm:plan-analyze → ...
        │                                (workflow/brainstorms/)
        │
        ├── A bug observation ─────────▶ /pm:bug-observe → /pm:bugfix
        │   (or for simple bugs)        /pm:bugfix directly
        │
        ├── An existing plan/spec ─────▶ /pm:plan-analyze → (/pm:prd-new →) /pm:epic-create → ...
        │
        └── A fully defined PRD ───────▶ /pm:epic-create → /pm:epic-decompose → ...
```

---

## File System Structure

All workflow artifacts live in `.claude/` for consistency and versioning:

```
.claude/
├── brainstorms/               # Design documents from /brainstorm
│   ├── brainstorm_*.md        # Active brainstorms
│   └── archive/               # Processed brainstorms
├── plans/                     # Implementation plans (from plan-analyze)
│   └── <feature-name>.md
├── prds/                      # Product Requirement Documents
│   └── <feature-name>.md
├── epics/                     # Epics and tasks
│   └── <feature-name>/
│       ├── epic.md            # Epic definition
│       ├── 001.md             # Task 1
│       ├── 002.md             # Task 2
│       └── validation-report.md
├── bugfixes/                  # Bug fix sessions
│   ├── observations/          # Bug reports from bug-observe
│   │   └── bug-<timestamp>.md
│   └── <session-id>/          # Bug fix session logs
│       ├── BUG-001.md
│       └── fix-report.md
└── commands/pm/               # All PM commands
```
