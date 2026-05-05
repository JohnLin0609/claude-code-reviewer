# /code-review — Claude Code Slash Command

A slash command for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that acts as a senior engineer reviewing your project. It builds a global model of how your code flows, hunts for **cross-function bugs** that only appear when functions interact, then sweeps for local issues — and waits for your approval before touching a single file.

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

The review runs in two stages, each enforced by writing artefacts to disk before moving on.

**Stage 1 — Workflow analysis** (the bugs that hide between functions):

| Phase | Output | What happens |
|-------|--------|-------------|
| **0 — Git Safety** | (commit) | Detects git, asks to initialise if missing, snapshots current state. |
| **1 — Inventory** | `01_inventory.md` | Maps entry points, modules, external boundaries, concurrency model. |
| **2 — Workflow Traces** | `02_traces.md` | Walks 2–3 user-visible paths end-to-end, recording entry/exit state at every step. |
| **3 — Contracts** | `03_contracts.md` | Extracts the implicit pre/post-conditions of each function in the traces. |
| **4 — Shared State** | `04_state.md` | Lists every shared mutable state with its writers, readers, and ordering guarantees. |
| **5 — Cross-Function Review** | `05_findings.md` | Hunts seven failure patterns + an adversarial pass against every step. |

**Stage 2 — Local sweep** (the bugs that fit inside one function):

| Phase | Output | What happens |
|-------|--------|-------------|
| **6 — Local Checklist** | (appends to `05_findings.md`) | Sweeps 8 categories: correctness, types, errors, resources, performance, security, structure, readability. |
| **7 — Write Plan** | `code_improve_plan.md` | Consolidates everything into a prioritised plan with 🔗 / 🔴 / 🟡 / 🟢 tiers. |
| **8 — Hand Back** | (summary) | Prints findings count and **stops**. Waits for your instruction. |
| **9 — Apply Fixes** | (commit) | Applies only the issues you approved. Snapshots before, commits after. |

All intermediate artefacts live in `_review_workspace/` so you can audit the reasoning behind every finding.

## Cross-Function Patterns Checked

Stage 1 hunts these specifically — they're the bugs a file-by-file reader misses:

- **A** Caller and callee disagree on a precondition
- **B** Step ordering / state-machine violation (paid order, no inventory)
- **C** Read-modify-write without atomicity
- **D** Error-handling discontinuity across an async boundary
- **E** Resource lifecycle leaks across a path
- **F** Two implementations of the same intent that have drifted apart
- **G** Implicit coupling via shared state

Plus an **adversarial pass** — for each step in each traced path, what happens if it's called twice, in parallel, with boundary input, partially fails, or runs out of order?

## Usage

Run inside a **Claude Code CLI** session:

```
/code-review              # review entire project
/code-review src/api      # review a specific subfolder or file
```

After the review completes, tell Claude what to fix:

```
"Fix all cross-function and critical issues"
"Fix issues C-1, 3, and L-5"
"Fix everything"
"Skip the medium issues and fix the rest"
```

Findings are numbered `C-<n>` (cross-function) and `L-<n>` (local) so you can target them precisely.

## Output

The plan written to `code_improve_plan.md`:

```markdown
## 🔗 Cross-function / workflow issues
- [ ] **[Pattern B]** `Path A` (checkout.js:42 ↔ inventory.js:88) —
      order marked paid before inventory decremented; on decrement failure,
      paid order with no hold → wrap both in a transaction

## 🔴 Critical — Fix Immediately
- [ ] **Error Handling** `auth.js:42` — exceptions swallowed silently → log and re-throw
- [ ] **Security** `db.js:17` — raw SQL string interpolation → use parameterized queries

## 🟡 High — Fix This Sprint
- [ ] **Performance** `search.js:88` — N+1 query inside loop → batch with a single JOIN
```

Items are marked `- [x]` as fixes are applied, so the file doubles as a progress tracker.

## Git Safety Net

Three commits per full cycle make every step individually revertible:

```
fix: apply approved changes from code-review   ← after fixes
chore: pre-fix snapshot                        ← right before fixes touch source
chore: pre-code-review snapshot                ← right before review begins
```

```bash
git diff HEAD~1             # see what the last step changed
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
