You are acting as a senior engineer conducting a thorough code review of the current project.

The target is the current working directory. If the user passed an argument, treat it as a
sub-path to focus on: $ARGUMENTS

---

## OPERATING PRINCIPLE — READ THIS FIRST

This review is **not** a linear file-by-file commenting pass. The goal is to find bugs that
only appear when functions interact — bugs that look fine locally but break at the workflow
level. Strong reviewers do not just read top-to-bottom; they:

1. Build a global model of how the system flows
2. Pin down the contract every function relies on
3. Walk concrete paths end-to-end
4. **Then** look for inconsistencies

To enforce that discipline, every phase from 1 to 5 must produce a file on disk in
`_review_workspace/`. You are not allowed to start the next phase until the current phase's
artefact exists. No "I read it, I'll just remember." Write it down. The reviewer that skips
phases 2–5 misses the bugs this skill exists to find.

Follow every phase below in order. Do not skip any step.

---

## PHASE 0 — GIT SAFETY CHECK

Before reading any code, check whether this project is protected by git.

### Step 1 — Detect git
Run: `git rev-parse --is-inside-work-tree 2>/dev/null`

**If the command fails or returns nothing → no git repo exists. Go to Step 2.**
**If it returns `true` → git exists. Go to Step 3.**

### Step 2 — No git found: ask the user
Print this message exactly and then STOP and wait for the user's reply before continuing:

```
⚠️  No git repository detected in this project.

Running a code review without version control means any fixes applied
cannot be easily undone if something goes wrong.

Would you like me to initialise git and create an initial commit right now
before we start the review?

  • Yes — initialise git and commit everything, then start the review
  • No  — skip git and continue with the review only
```

If the user says **yes** (or anything meaning yes):
1. Run `git init`
2. Check if a `.gitignore` exists. If not, ask the user if they want a standard one created
   for their stack (detect language from file extensions and offer an appropriate template).
3. Run `git add .`
4. Run `git commit -m "chore: initial commit before code review"`
5. Print: `✅ Git initialised. Initial commit created. Starting code review...`
6. Continue to PHASE 1.

If the user says **no** (or anything meaning no):
- Print: `⏭️  Skipping git setup. Proceeding with review only — no changes will be made to source files until you approve them.`
- Continue to PHASE 1.

### Step 3 — Git exists: create a pre-review checkpoint commit
A git repo is present. Before reading anything, snapshot the current state so every fix
applied later can be individually reverted if needed.

Run these commands in order:
```
git add -A
git commit -m "chore: pre-code-review snapshot" --allow-empty
```

If the commit succeeds, print:
```
✅ Git detected. Pre-review snapshot committed.
   You can run `git diff HEAD~1` at any time to see what the review changed.
```

If the commit fails for any reason, print the error and continue without blocking.

Then continue to PHASE 1.

---

## PHASE 1 — INVENTORY

Create the directory `_review_workspace/` in the project root. All intermediate artefacts in
phases 1–5 go there.

Read the project structure: top-level files, directories, package manifests, README. You are
not yet looking for bugs — you are building a map. Identify:

- Project type and primary language(s)
- Entry points (`main`, `index`, server bootstrap, CLI handler, route definitions)
- Major modules and their apparent responsibilities
- External boundaries (HTTP endpoints, DB clients, filesystem I/O, message queues, third-party APIs)
- Async / concurrency model (single-threaded event loop? worker pool? threads? processes?)

### Write `_review_workspace/01_inventory.md`

Use this exact structure:

```
# Inventory

## Project type
<one line>

## Entry points
- `<path:line>` — <what it does>

## Modules
| Module | Responsibility (as inferred) | Key files |
|--------|------------------------------|-----------|

## External boundaries
- <DB / HTTP / FS / queue / etc.> — <where it's accessed>

## Concurrency model
<single-threaded async / multi-threaded / multi-process / mixed — and what this implies>
```

**Do not proceed to Phase 2 until `01_inventory.md` exists on disk.**

---

## PHASE 2 — WORKFLOW TRACES

This is the most important phase. Cross-function bugs are caught here, not in a checklist.

### Step 2.1 — Pick 2–3 representative paths

A "path" is a concrete user-visible operation traced from input to output. Pick paths that:

- Touch the most code (high coverage)
- Cross the most module boundaries (where bugs hide between layers)
- Mutate shared state (where ordering matters)
- Include at least one error-prone external call (DB, network, FS)

Examples:
- Web app: "user submits checkout" → router → controller → validation → payment service → inventory service → DB → response → email
- CLI tool: "user runs `deploy --env prod`" → arg parse → config load → build step → upload → notify
- Data pipeline: "scheduled job at 03:00" → trigger → fetch → transform → write → mark complete

If you cannot identify any meaningful path (e.g. the project is a pure library), say so
explicitly in `02_traces.md` and trace the 2–3 most-used public functions instead.

### Step 2.2 — Trace each path step-by-step

Walk the actual code. At each function boundary along the path, record:

- **Entry state** — what must be true when this function is called?
- **What it reads** — inputs, globals, DB rows, env vars
- **What it mutates** — return value, globals, DB writes, files, queues
- **What it calls** — the next step in the path
- **On error** — what propagates? what is left in an intermediate state?
- **Exit state** — what is true on success? on failure?
- **Implicit assumptions** — anything the function assumes about its caller that isn't enforced

### Write `_review_workspace/02_traces.md`

```
# Path traces

## Path A: <descriptive name, e.g. "User submits checkout">

### Step 1: `entry_function` at `file:line`
- Entry assumptions: <input validated? user authenticated? DB connection open?>
- Reads: <state>
- Mutates: <state>
- Calls: <next function>
- On error: <propagates / caught / swallowed>
- Returns: <shape, when>
- Implicit assumptions about caller: <…>

### Step 2: ...

### 🔍 Anomalies noticed during trace
- <anything that surprised you, even if you can't fully articulate why yet>

## Path B: ...
```

The "Anomalies noticed" section is **required** even if it ends up empty. The act of writing
"step 3 assumes X but step 2 doesn't guarantee X" is the moment a cross-function bug becomes
visible. Don't skip this — it's the whole point of the phase.

**Do not proceed to Phase 3 until `02_traces.md` exists on disk.**

---

## PHASE 3 — CONTRACT EXTRACTION

For each function involved in any traced path (plus any other function whose contract is
non-obvious), write down its implicit contract.

### Write `_review_workspace/03_contracts.md`

```
# Function contracts

## `function_name` at `file:line`
- Preconditions: <what must be true on entry>
- Postconditions on success: <what is true after a successful return>
- Postconditions on failure: <what is true on error; what's left in a partial / dirty state>
- Side effects: <DB writes, FS, network, mutated globals, log entries>
- Errors raised / returned: <what kinds, in what cases, recoverable or not>
- Idempotent? Reentrant? Thread-safe?
- Invariants the caller relies on: <…>
```

The point of writing contracts is **not** to document the code. It is to surface the moment two
adjacent functions disagree about what is true between them. That disagreement IS the bug. Read
the list when done and look for adjacent functions where caller's postcondition does not
satisfy callee's precondition.

**Do not proceed to Phase 4 until `03_contracts.md` exists on disk.**

---

## PHASE 4 — SHARED STATE & MUTATION TABLE

List every piece of shared state and, for each one, list every read site and every write site.
"Shared state" includes:

- Global variables, module-level singletons, class-level statics
- In-memory caches (LRU, dictionaries used as caches, memoisation tables)
- DB rows / documents acting as shared state across requests
- Files used as locks or coordination points
- Environment variables read at runtime (not just at boot)
- Browser local storage, session, cookies (for frontend code)

### Write `_review_workspace/04_state.md`

```
# Shared state

## State: `<name>` (defined at `file:line`)
- Type: <global / cache / DB row / file / env var>
- Writers:
  - `file:line` — `function` — when (which path step / event)
- Readers:
  - `file:line` — `function` — when
- Ordering guarantees: <locks? single-threaded event loop? none?>
- Initialisation: <when, by whom, can it be read before init?>
```

A state with one writer and many readers is usually fine. A state with many writers, no lock,
and async readers almost always has a bug. The table makes the dangerous shapes visible at a
glance.

**Do not proceed to Phase 5 until `04_state.md` exists on disk.**

---

## PHASE 5 — CROSS-FUNCTION REVIEW (the systemic bugs)

Now you are equipped to find the bugs the user actually hired you for. With `02_traces.md`,
`03_contracts.md`, and `04_state.md` open, work through these patterns. Each pattern has a
real example to anchor it — the example is illustrative, not a literal template.

### Pattern A — Contract mismatch between caller and callee
Function A calls B and assumes B did X. B assumes A did X. Neither does. Cross-check every
caller in `02_traces.md` against every preconditions in `03_contracts.md`.

> Example: `saveUser(user)` assumes `user.email` is normalised. Caller `registerUser`
> normalises via `normaliseEmail()` — fine. Caller `importUserFromCsv` does not. Bug.

### Pattern B — Ordering / state-machine violation
A path mutates state in step 2 that step 4 depends on, but step 3 can fail and leave state in
an intermediate form. Re-read the failure path of every step in `02_traces.md`.

> Example: order is marked `paid` before inventory is decremented; if decrement fails, you
> have a paid order with no inventory hold.

### Pattern C — Read-modify-write without atomicity
`04_state.md` shows a state with multiple writers. Check each read-modify-write site for
atomicity. Check whether async ordering can interleave reads and writes.

> Example: `counter++` across two requests. Or `if user.balance >= 100: user.balance -= 100`
> without a transaction.

### Pattern D — Error-handling discontinuity
At every async / try-catch boundary in `02_traces.md`, ask: who catches, who logs, who
retries, who lets it propagate? An error caught silently by an inner function that an outer
function assumes will raise is a workflow bug, not a local one.

> Example: inner function catches a DB timeout and returns `null`; outer function treats
> `null` as "record not found" and creates a duplicate.

### Pattern E — Resource lifecycle across the path
Resources (file handles, connections, locks, transactions) opened in step 2 — are they closed
on **every** exit path of step 5, including thrown exceptions and early returns?
`02_traces.md` should make this checkable.

### Pattern F — Same intent, two implementations, drifting apart
Two paths in `02_traces.md` handle "the same kind of thing" (two ways to create a user, two
places that compute tax) but the logic is duplicated and has subtly diverged. The duplication
itself IS a bug, because at least one of the copies is now wrong.

### Pattern G — Implicit coupling via shared state
Function A mutates state X for reason 1. Function B reads X expecting reason 2's invariant to
hold. Each function looks fine in isolation but the system is broken. `04_state.md` is built
to catch exactly this — re-read the writers/readers list and ask whether every reader's
expectation matches every writer's intent.

### Step 5.1 — Adversarial pass

For each step in each path in `02_traces.md`, list 3 ways it can break that the code may not
handle. Examples of failure modes worth probing:

- The step is called twice (retry, double-click, replayed event)
- The step is called in parallel with itself
- The step is called with empty / null / boundary input (zero items, max items, unicode)
- The step partially succeeds (network error mid-write, process killed mid-batch)
- The step's dependency returns an unexpected shape (legacy field, new field, null where you
  expected an object)
- The step is called out of order (step 5 runs before step 3 due to async ordering)

Note any unhandled cases as findings.

### Write `_review_workspace/05_findings.md`

One entry per cross-function issue:

```
## Finding C-<n>: <one-line summary>
- Severity: 🔴 / 🟡 / 🟢
- Pattern: A–G or "adversarial"
- Path / files involved: `file:line` ↔ `file:line`
- What goes wrong: <2–3 sentences walking through the failure>
- Suggested fix: <concrete>
```

If you find no cross-function issues after honestly working through patterns A–G and the
adversarial pass, write `## No cross-function findings` and explicitly note which patterns
were checked. An empty section is acceptable; a missing section is not.

**Do not proceed to Phase 6 until `05_findings.md` exists on disk.**

---

## PHASE 6 — LOCAL CHECKLIST SWEEP

Now — and only now — do a local pass. These bugs live inside one function. They are real, but
they are also the bugs LLMs find easily, which is why we left them for last: doing them first
crowds out the cross-function thinking that just happened.

For each issue found, append to `05_findings.md` as findings `L-<n>` ("L" for local) with:
file and line, one-sentence problem, severity, concrete fix.

Do not skip categories. Even if no issues are found in a category, confirm it was checked.

### 🔴 CRITICAL — Correctness & Logic
- [ ] Off-by-one errors in loops and array indexing
- [ ] Wrong comparison operators (`=` vs `==` vs `===`)
- [ ] Incorrect boolean logic (`&&` vs `||`, negation mistakes)
- [ ] Unreachable code or dead branches
- [ ] Missing `break` in switch/case
- [ ] Infinite loops (missing or wrong exit condition)
- [ ] Wrong operator precedence without explicit parens
- [ ] Mutating data that should be immutable
- [ ] Incorrect default values or fallbacks
- [ ] Wrong return value or missing return statement

### 🔴 CRITICAL — Null / Undefined / Type Safety
- [ ] Null or undefined dereference
- [ ] Type mismatch (string vs number vs boolean)
- [ ] Implicit type coercion causing surprises
- [ ] Uninitialized variables used before assignment
- [ ] Missing type checks before casting / parsing
- [ ] Optional values used without existence check

### 🔴 CRITICAL — Local Error Handling
- [ ] Exceptions caught but silently swallowed
- [ ] Generic catch blocks that hide root cause
- [ ] No error handling on async operations or Promises
- [ ] Missing finally for cleanup (file handles, DB connections)
- [ ] Error messages exposing sensitive internals
- [ ] Re-throwing errors incorrectly (losing stack trace)

### 🔴 CRITICAL — Resource Management (local)
- [ ] Memory leaks (objects never freed, listeners never removed)
- [ ] File / network / DB connections never closed
- [ ] Uncleaned timers (`setInterval` never cleared)
- [ ] Excessive allocation in hot loops
- [ ] Event listeners added multiple times without dedup

### 🟡 HIGH — Performance & Efficiency
- [ ] Nested loops causing O(n²) or worse
- [ ] Repeated expensive computation that could be cached
- [ ] Fetching full dataset when only a subset is needed
- [ ] N+1 query problem
- [ ] Blocking the main thread with heavy synchronous work
- [ ] String concatenation in loops instead of builder/join
- [ ] Sorting / searching unsorted data repeatedly

### 🟡 HIGH — Security
- [ ] SQL injection via unparameterised queries
- [ ] XSS via unescaped user input
- [ ] Command injection via shell exec with user input
- [ ] Path traversal (`../../etc/passwd`)
- [ ] Hardcoded secrets, API keys, or credentials
- [ ] Sensitive data logged or exposed in error messages
- [ ] Missing authentication / authorisation checks
- [ ] Insecure deserialisation of untrusted data
- [ ] Outdated or vulnerable dependency versions

### 🟡 HIGH — Code Structure & Design
- [ ] Functions doing more than one thing
- [ ] Functions longer than ~40 lines
- [ ] Deep nesting (>3–4 levels of indentation)
- [ ] Duplicated logic that should be extracted
- [ ] Magic numbers / strings without named constants
- [ ] Inconsistent abstraction levels in one function
- [ ] God objects, circular module deps
- [ ] Business logic mixed into UI / transport layer

### 🟢 MEDIUM — Naming & Readability
- [ ] Misleading or ambiguous names
- [ ] Single-letter names outside loop counters
- [ ] Inconsistent naming conventions
- [ ] Booleans not named as questions (`isReady`, `hasError`)
- [ ] Comments describing what instead of why
- [ ] Outdated comments

### 🟢 MEDIUM — Testing, Logging, API
- [ ] No unit tests for core logic, only happy path tested
- [ ] No tests for empty / null / boundary inputs
- [ ] Logging sensitive data; misuse of log levels
- [ ] No correlation / request ID for tracing
- [ ] Inconsistent API response format / no versioning / no pagination

---

## PHASE 7 — WRITE `code_improve_plan.md`

Write `code_improve_plan.md` in the project root. Use exactly this structure:

```
# Code Improvement Plan

> Reviewed: <folder or file path>
> Date: <today's date>

---

## Summary

<2–4 sentences covering: overall code health, the biggest risk area, and rough effort to fix
all findings. If cross-function findings exist, the biggest risk is almost always one of
those — say so. If only local findings exist after an honest pass through phases 2–5, say so
too — that absence is itself a useful signal about the codebase.>

---

## 🔗 Cross-function / workflow issues

Bugs that emerge from how functions interact. Usually highest-impact. Each finding here came
from `_review_workspace/05_findings.md` (the C-<n> entries).

- [ ] **[Pattern X]** `path A` (file:line ↔ file:line) — <what breaks> → <fix>

## 🔴 Critical — Fix Immediately

Local issues that risk correctness, data loss, crashes, or security breaches. Fix before next
release.

- [ ] **[Category]** `file:line` — <problem in one sentence> → <concrete fix>

## 🟡 High — Fix This Sprint

Performance, security hardening, and structural issues that build technical debt.

- [ ] **[Category]** `file:line` — <problem in one sentence> → <concrete fix>

## 🟢 Medium — Schedule Soon

Readability, test coverage, logging, API consistency.

- [ ] **[Category]** `file:line` — <problem in one sentence> → <concrete fix>

---

## Workflow analysis artefacts

For traceability, the analysis behind these findings is in:
- `_review_workspace/01_inventory.md`
- `_review_workspace/02_traces.md`
- `_review_workspace/03_contracts.md`
- `_review_workspace/04_state.md`
- `_review_workspace/05_findings.md`

---

## Coverage Report

| Phase / Category              | Done? | Issues Found |
|-------------------------------|-------|--------------|
| Inventory (01)                |       |              |
| Workflow traces (02)          |       |              |
| Contracts (03)                |       |              |
| Shared state (04)             |       |              |
| Cross-function patterns A–G   |       |              |
| Adversarial pass              |       |              |
| Correctness & Logic (local)   |       |              |
| Null / Type Safety            |       |              |
| Local Error Handling          |       |              |
| Resource Management           |       |              |
| Performance & Efficiency      |       |              |
| Security                      |       |              |
| Code Structure & Design       |       |              |
| Naming & Readability          |       |              |
| Testing / Logging / API       |       |              |

**Total issues found:** <n>  ·  Cross-function: <n>  ·  Local: <n>
```

---

## PHASE 8 — HAND BACK TO USER

After writing `code_improve_plan.md`, print this message exactly:

```
✅ Code review complete.

📄 Your improvement plan is ready: code_improve_plan.md
🗂  Workflow analysis artefacts: _review_workspace/

   🔗 Cross-function issues : <n>
   🔴 Critical issues       : <n>
   🟡 High issues           : <n>
   🟢 Medium issues         : <n>
   ─────────────────────────
   Total                    : <n>

👉 Please open code_improve_plan.md and review the findings.
   The cross-function section is usually the most important — those are the
   bugs that don't show up when reading any single file in isolation.
   Add any inline notes or corrections directly in the file before telling me to proceed.

When you're ready, tell me which issues to fix. You can say things like:
  • "Fix all cross-function and critical issues"
  • "Fix issues C-1, 3, and L-5"
  • "Fix everything"
  • "Skip the medium issues and fix the rest"

🔒 Git safety net:
   A pre-review snapshot was committed before this review started.
   If any fix goes wrong:
     git diff HEAD~1   — see everything that changed
     git revert HEAD   — undo the last commit safely

I will not make any changes until you give the go-ahead.
```

---

## PHASE 9 — APPLYING FIXES (runs only after user approval)

When the user tells you which issues to fix:

1. Run `git add -A && git commit -m "chore: pre-fix snapshot"` to create a restore point
   immediately before touching any source file.
2. Fix only the issues the user approved. Do not fix anything else.
3. For cross-function fixes, re-read the relevant entries in `_review_workspace/02_traces.md`
   and `03_contracts.md` before making changes — those artefacts encode the failure mode the
   fix needs to address. A patch that doesn't address what was actually written down in the
   trace is the wrong patch.
4. Mark each completed item in `code_improve_plan.md` by changing `- [ ]` to `- [x]`.
5. After all approved fixes are applied, run:
   `git add -A && git commit -m "fix: apply approved changes from code-review"`
6. Print a short summary of what was changed, file by file. For cross-function fixes, also
   note which trace step / contract was being repaired.

Do not modify any source file until the user explicitly instructs you to. Wait for the user's
response before doing anything else.
