# /code-review — Claude Code Slash Command

A slash command for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that acts as a senior engineer reviewing your project. It reads deeply, checks 13 categories (~100 checklist items), writes a prioritised improvement plan, and waits for your approval before touching a single file.

## Quick Start

```bash
# Install globally (available in every project)
curl -o ~/.claude/commands/code-review.md \
  --create-dirs \
  https://raw.githubusercontent.com/JohnLin0609/claude-code-reviewer/main/code-review.md

# Then inside any Claude Code session:
/code-review
```

Or install for a single project only:

```bash
curl -o .claude/commands/code-review.md \
  --create-dirs \
  https://raw.githubusercontent.com/JohnLin0609/claude-code-reviewer/main/code-review.md
```

No restart required. Claude Code picks up the command immediately.

## How It Works

| Phase | What happens |
|-------|-------------|
| **0 — Git Safety** | Detects git. If missing, asks to initialise. If present, commits a pre-review snapshot. |
| **1 — Read Deeply** | Traces entry points, data flow, shared state, async boundaries, and external calls. |
| **2 — Systematic Review** | Checks 13 categories. No category is skipped. |
| **3 — Write Plan** | Produces `code_improve_plan.md` with every finding prioritised into 🔴 / 🟡 / 🟢 tiers. |
| **4 — Hand Back** | Prints a summary and **stops**. Waits for your instruction before making any change. |
| **5 — Apply Fixes** | Applies only the issues you approved, marks them done, leaves changes **uncommitted** for your review. |

## Review Categories

| Tier | Categories |
|------|-----------|
| 🔴 Critical | Correctness & Logic, Null / Type Safety, Concurrency & Race Conditions, Error Handling, Resource Management |
| 🟡 High | Performance & Efficiency, Security, Code Structure & Design, Dependency & Environment |
| 🟢 Medium | Naming & Readability, Testing & Coverage, Logging & Observability, API & Interface Design |

## Usage

Run inside a **Claude Code CLI** session:

```
/code-review              # review entire project
/code-review src/api      # review a specific subfolder or file
```

After the review completes, tell Claude what to fix:

```
"Fix all critical issues"
"Fix issues 1, 3, and 5"
"Fix everything"
"Skip the medium issues and fix the rest"
```

## Output

A `code_improve_plan.md` file is written to the project root:

```markdown
## 🔴 Critical — Fix Immediately
- [ ] **Error Handling** `auth.js:42` — exceptions swallowed silently → add logging and re-throw
- [ ] **Security** `db.js:17` — raw SQL string interpolation → use parameterized queries

## 🟡 High — Fix This Sprint
- [ ] **Performance** `search.js:88` — N+1 query inside loop → batch with a single JOIN
```

Items are marked `- [x]` as fixes are applied, so the file doubles as a progress tracker.

## Git Safety Net

Every run creates a snapshot commit so fixes can be individually reverted:

```bash
git diff HEAD~1             # see what the review changed
git restore .               # discard uncommitted changes
git revert HEAD             # undo last commit safely
```

## Updating

Re-run the install command to pull the latest version:

```bash
curl -o ~/.claude/commands/code-review.md \
  --create-dirs \
  https://raw.githubusercontent.com/JohnLin0609/claude-code-reviewer/main/code-review.md
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- Git (optional but recommended — the command offers to set it up if missing)

## License

[MIT](LICENSE)
