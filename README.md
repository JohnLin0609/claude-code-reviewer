# /code-review — Claude Code Slash Command

A Claude Code slash command that acts as a senior engineer performing a thorough,
systematic code review of your project. It reads deeply, checks 13 categories of
problems, writes a prioritised improvement plan, and waits for your approval before
touching a single file.

---

## What It Does

```
/code-review
```

Runs five phases automatically:

| Phase | What happens |
|-------|-------------|
| **0 — Git Safety** | Detects git. If missing, asks to initialise. If present, commits a pre-review snapshot. |
| **1 — Read Deeply** | Traces entry points, data flow, shared state, async boundaries, and external calls. |
| **2 — Systematic Review** | Checks 13 categories across 60+ checklist items. No category is skipped. |
| **3 — Write Plan** | Produces `code_improve_plan.md` with every finding prioritised into 🔴 / 🟡 / 🟢 tiers. |
| **4 — Hand Back** | Prints a summary and stops. Waits for your instruction before making any change. |

When you approve fixes, Phase 5 applies only the issues you selected, marks them as done
in the plan file, and leaves the changes **uncommitted** so you can review the diff first.

---

## Checklist Categories

### 🔴 Critical — Fix Immediately
- Correctness & Logic
- Null / Undefined / Type Safety
- Concurrency & Race Conditions
- Error Handling
- Resource Management

### 🟡 High — Fix This Sprint
- Performance & Efficiency
- Security
- Code Structure & Design
- Dependency & Environment

### 🟢 Medium — Schedule Soon
- Naming & Readability
- Testing & Coverage
- Logging & Observability
- API & Interface Design

---

## Installation

**Global** — available in every project on your machine:

```bash
mkdir -p ~/.claude/commands
cp code-review.md ~/.claude/commands/code-review.md
```

**Project-only** — available only inside a specific project:

```bash
mkdir -p .claude/commands
cp code-review.md .claude/commands/code-review.md
```

No restart required. Claude Code picks up the command immediately.

---

## Usage

This is a slash command for the **Claude Code CLI**. Run it inside the Claude Code terminal session:

```
/code-review
```

Review a specific subfolder or file only:

```
/code-review src/api
```

---

## The Full Workflow

```
/code-review
    │
    ├── No git? → Asks to initialise → git init + initial commit
    │
    ├── Git exists? → Commits pre-review snapshot automatically
    │
    ├── Reads entire codebase deeply
    │
    ├── Runs all 13 checklist categories
    │
    ├── Writes code_improve_plan.md
    │
    └── Prints summary + STOPS ◄── you are here
              │
              │  You review the plan. Tell Claude what to fix:
              │  "Fix all critical issues"
              │  "Fix issues 1, 3, and 5"
              │  "Fix everything"
              │
              └── Applies only approved fixes (uncommitted)
                        │
                        └── You run: git diff
                                  git add -A && git commit -m "..."
```

---

## Git Safety Net

Every run produces a commit trail you can navigate:

```bash
git log --oneline
# a3f9c12  fix: apply approved changes from code-review
# 8b2e441  chore: pre-code-review snapshot
# f1a2b87  chore: initial commit before code review
```

Useful commands after a review:

```bash
git show                    # see what the last commit changed
git diff HEAD~1             # compare current state to before review
git restore .               # discard all uncommitted changes
git revert HEAD             # undo last commit safely
```

---

## Output: code_improve_plan.md

After every run a `code_improve_plan.md` is written to the project root:

```markdown
# Code Improvement Plan

> Reviewed: ./src
> Date: 2026-04-08

## Summary
Overall the codebase is in fair health...

## 🔴 Critical — Fix Immediately
- [ ] **Error Handling** `auth.js:42` — exceptions swallowed silently → add logging and re-throw
- [ ] **Security** `db.js:17` — raw SQL string interpolation → use parameterized queries

## 🟡 High — Fix This Sprint
- [ ] **Performance** `search.js:88` — N+1 query inside loop → batch with a single JOIN

## 🟢 Medium — Schedule Soon
- [ ] **Naming** `utils.js:5` — function named `data()` → rename to `fetchUserData()`

## Checklist Coverage Report
| Category                      | Status | Issues Found |
|-------------------------------|--------|--------------|
| Correctness & Logic           | ✅     | 0            |
| ...                           | ...    | ...          |

**Total issues found:** 4
```

Items are marked `- [x]` as fixes are applied, so the file doubles as a progress tracker.

---

## Updating the Command

When a new version of `code-review.md` is released, replace the old file:

```bash
# Global install
cp code-review.md ~/.claude/commands/code-review.md

# Project install
cp code-review.md .claude/commands/code-review.md
```

No restart needed. The next `/code-review` run uses the new version.

---

## Re-running After Fixes

Running `/code-review` multiple times on the same project is intentional and expected.
Each run is a fresh analysis of the current state of the code. As you fix issues,
the 🔴 critical count should drop toward zero across runs. The goal is not "zero findings"
— it is "zero critical findings."

To compare two runs, rename the plan before re-running:

```bash
mv code_improve_plan.md code_improve_plan_v1.md
/code-review
# now diff the two files to see what changed
```

---

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed
- Git (optional but strongly recommended — the command will offer to set it up if missing)

---

## Files

| File | Purpose |
|------|---------|
| `code-review.md` | The slash command — copy this to `.claude/commands/` |
| `SKILL.md` | Reference document describing the review methodology |

---

## Inspired By

Workflow design based on
[How I Use Claude Code](https://boristane.com/blog/how-i-use-claude-code/) by Boris Tane —
specifically the principle of **never letting Claude write or change anything until a written
plan has been reviewed and approved**.
