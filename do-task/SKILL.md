---
name: do-task
description: Use this skill whenever working on any codebase that has been indexed with a modules documentation system — when fixing bugs, implementing features, explaining how something works, tracing a flow across files, refactoring, or answering any question about the architecture, business domain, specific files, or "where is X defined / who calls Y". The skill auto-detects the modules folder by checking common paths (`docs/modules/`, `*-docs/modules/`, `modules/`, `documentation/modules/`), reads `00-index.md` to identify relevant modules, then reads module docs + actual code BEFORE acting. This is dramatically faster than grepping cold. It works as a planner *and* executor: it produces a plan + checklist, persists it to a single task file under the docs folder (`tasks/`), tracks execution against it, and verifies every item with evidence before reporting done. After making changes, the skill updates module docs to keep them in sync so they don't rot. If no modules folder is found, the skill asks whether docs live at a non-standard path or offers to invoke `index-codebase` on the spot, then continues the task seamlessly. Use this BEFORE touching code in any indexed project.
---

# Do-Task — Module-First Planner & Executor (Project-Agnostic)

## Philosophy

**Docs are the map. Code is the territory.** Read the map before moving, update the map after changing the terrain — but **never confuse the map for the territory.** The map tells you where to start; correctness comes from walking the actual code to its real boundary. On large, complex codebases with hidden flows, the map is necessary but never sufficient — verify everything against the code (Step 4 Completeness protocol).

This skill applies to any codebase indexed by `index-codebase` (or an equivalent process): a `modules/` folder with `00-index.md` plus N module docs in a standard format.

**Plan, then execute — against a written contract.** do-task is a planner *and* an executor: it turns the request into a plan + checklist, **persists that to a single task file** under the docs folder (so the contract survives context loss, compaction, or a brand-new session), executes against it, and reconciles every item with evidence before reporting done. The task file is a working record you own — move, archive, or delete it freely. (The "docs are reference, not journal" rule governs `modules/` docs only; task files are dated working records by design, kept in a separate `tasks/` folder.)

## When to trigger

- User describes a bug / pastes an error message
- User wants to implement a new feature
- User asks "how does X work", "what's the flow for Y", "where is Z used", "who calls this function"
- User wants to refactor / split a module / merge modules
- Any coding task in a project that has modules docs

## Workflow

### Step 0 — Locate the modules folder

Auto-discover by checking these paths (in order) for `00-index.md`:

1. `docs/modules/`
2. `*-docs/modules/` (glob — pattern like `driver-ms-docs/modules/`)
3. `modules/`
4. `documentation/modules/`

If multiple matches, prefer the more specific one (project-named docs folder beats generic `modules/`). If still unclear, ask the user once.

If none match, run this 2-step fallback:

1. **Ask the user whether docs live at a non-standard path** (e.g., `wiki/`, `.docs/`, `internal-docs/`, a sibling repo, etc.). If yes → take that path as `modules_path` and continue. Verify `00-index.md` exists there; if not, treat as "no docs" and go to step 2.

2. **If there are genuinely no docs**, ask: *"This project has no modules docs yet. Want me to index it now? (invokes the `index-codebase` skill, takes a few minutes depending on repo size.)"*
   - **Yes** → invoke the `index-codebase` skill via the `Skill` tool. After it finishes, re-run Step 0 to discover the freshly-created `modules_path`, then continue with the user's original task seamlessly.
   - **No** → STOP. Do not fall back to ad-hoc grepping. This skill depends on the docs.

Save the discovered `modules_path` for the rest of the task.

Also derive two more paths:
- `docs_root` = the parent of `modules_path` (e.g. `docs/modules/` → `docs/`; `driver-ms-docs/modules/` → `driver-ms-docs/`; a bare `modules/` at repo root → repo root).
- `tasks_path` = `{docs_root}/tasks/` — where this task's plan file lives (create the folder if missing). It sits **beside** `modules/`, never inside it, so Step 0's auto-discovery never mistakes a task plan for a module doc.

### Step 1 — Understand the user's request

Before reading anything, identify (privately):

- **Intent**: `fix-bug` / `feature` / `question` / `refactor`
- **Domain keywords**: business terms (e.g., "JWT login cross-tenant", "invoice PDF Japanese font", "approval merge")
- **Scope**: single module or cross-module?

Infer from keywords if the user wasn't explicit. If genuinely ambiguous, ask one short clarifying question — don't guess blindly.

### Step 2 — Read 00-index.md

Read `{modules_path}/00-index.md` first. It lists all modules, their domain keywords, DB tables, and cross-references.

Identify 1–3 relevant module(s). For cross-module cases, list them all.

### Step 3 — Read the relevant module docs

For each identified module, read the **entire** file `{modules_path}/{NN}-{slug}.md`. Pay attention to:

- **KEY FEATURES** — business overview
- **BACKEND FILES** / **FRONTEND FILES** — concrete file paths
- **API ENDPOINTS** — method/path/payload/response
- **DATABASE** — schema, enum values, foreign keys
- **WORKFLOW** — business flow
- **TRIGGERS & SIDE EFFECTS** — inbound/outbound hidden flows (events, jobs, observers, cross-module calls); this is your starting map for the Step 4 hidden-dispatch sweep — but verify it against code, it may be incomplete or stale
- **NOTES / GOTCHAS** — edge cases, deadcode flags, business rules that aren't obvious from code

Note which important functions live in which files, and which business rules apply to the current task.

### Step 4 — Read the actual code (selective by file, exhaustive by flow)

Docs tell you **where to start**; they do not bound what you must read. Use the file paths in the docs as entry points, then **follow the flow to its real boundary — across module lines — until nothing is left unexplained.** "Selective" means you don't blindly read the whole module folder; it never means you stop before the flow is fully traced.

Start from the natural entry point for the intent, then keep going:

- Bug at endpoint → controller method → service → model → **every other consumer of the changed symbol** (Completeness protocol below)
- Feature touching UI → FE page + components + the BE endpoint + backend service + **downstream side effects** (events, jobs, notifications)
- Question about a workflow → trace from entry point through services and **every branch**, not just the happy path
- Refactor → the "before" shape **plus every caller and every override**

While reading, hold on to: what each function does, its inputs/outputs, and side effects (notifications dispatched, jobs queued, recalc triggered, DB writes, events emitted).

If docs drift from code (e.g., docs mention a method that no longer exists, or code has a method docs missed) → flag it in the report. Don't trust docs blindly. Note these for the doc update in Step 7.

**Staleness check (cheap — do it once per relevant module, before trusting the doc).** Each doc footer carries `Synced: {sync-spec}` and `Format: v{N}`. The sync-spec is **one or more entries** — it adapts to the project's repo layout (single repo, split BE/FE, monorepo, or none). For **each** entry:
- `repo@sha` → locate `repo` (a path relative to the project root), run `git diff --name-only {sha} HEAD` inside it, and intersect with the module's files that live under that repo. Any overlap → the doc is possibly stale **for those files**: trust the code over the doc, mark the module for a Step 7 re-sync.
- `group#hash` (non-git files) → recompute the hash and compare; if it differs → stale.
- unresolvable repo, `n/a`, or missing → skip that entry silently (graceful degradation — fall back to the drift-flag above).

Check what you can and skip what you can't — a multi-repo module where only one repo is reachable is still partially verifiable. If `Format:` is absent or differs from what this skill expects, note the docs may predate the current format and a re-index could help.

#### Completeness protocol — trace to the real boundary

**Mandatory for `fix-bug` / `feature` / `refactor`, and for any `question` about how a flow works / where something is used / who calls what** (skip only for pure-mechanical edits — typo, comment, formatting — and trivial lookups that make no claim about behavior). On a large codebase the behavior you care about is usually driven from somewhere the module doc does not list — because **a module is a partition by domain, not by execution flow**: the files a given flow actually touches routinely live outside it (cross-module callers, hidden-dispatch handlers, shared/base code). The module's own files are the START, not the whole picture. Before you answer a flow/usage question — or write the Step 5 plan for a change — you MUST:

1. **Enumerate every usage of every symbol in scope.** For each function / method / class / route / event / config-key / DB-column you'll touch, grep the **entire codebase — every repo/root in the project, not just the module** (in a multi-repo project that means BE and FE both) for its name and aliases. List all call sites. A change isn't understood until every caller is accounted for. "No other callers" is a valid claim *only after* you grepped — state the pattern you searched.

2. **Sweep the hidden-dispatch surface.** Indirect flows never show up by reading the controller top-to-bottom. Actively grep for each mechanism the stack uses:
   - Events / listeners / subscribers — both emit and handle sites
   - Queued jobs / async workers / message-queue handlers
   - Model observers / lifecycle hooks (creating/saved/deleting…)
   - Scheduled tasks / cron / command bus
   - Middleware / interceptors / filters / guards
   - DI bindings / service providers / factories (interface → concrete impl)
   - Config- or string-driven dispatch (route names, class-strings, `app(x)` / container `make`, reflection, dynamic / `__call` methods)
   - Polymorphic / dynamic relations, morph maps
   - Inheritance, method overrides, traits/mixins (the live method may sit on a parent or sibling)
   - Webhooks / external callbacks / API clients
   - Feature flags / config branches that gate the flow

3. **Read what you found — never infer.** Open each non-trivial call site and confirm what it actually does. Never conclude a root cause, a behavior, or "nothing else is affected" from a doc or from a function's name.

4. **Default to cross-module blast radius.** If a changed symbol is referenced from another module, that module is in scope — read it before concluding.

Any assumption you could not verify goes verbatim into the Step 5 plan (or your answer, for a question) as `unverified: …`. Do not present a guess as a fact.

### Step 5 — Plan + checklist → write the task file (BEFORE acting — hard stop for review)

Build the plan in the shape below (in the user's locale; headings are placeholders), **write it to `{tasks_path}/{task-file}`** (one file = plan + checklist + later the reconciliation), and present it — or a tight summary plus the file path — to the user. Writing the file *now* means the contract survives even if the session is lost during review.

**Task-file naming.** `{task-file}` = `{YYYY-MM-DD}-{intent}-{slug}-implementation-plan.md`
- `{YYYY-MM-DD}` — today's date (chronological ordering + uniqueness).
- `{intent}` — maps the Step 1 intent: `fix-bug`→`fix`, `feature`→`feat`, `refactor`→`refactor` (pure-question tasks don't create a file).
- `{slug}` — 3–6 words from the task in **kebab-case ASCII**: lowercase, hyphen-separated, diacritics stripped (Vietnamese → no dấu), only `[a-z0-9-]`, ≤ ~40 chars.
- `-implementation-plan` — fixed suffix, so the file's purpose is obvious at a glance in the folder.
- Collision (a file with that name already exists): append `-2`, `-3`, … to the slug.
- Status is **not** in the filename — it lives in the file's `Status:` field, so the file never needs renaming.
- Examples: `2026-06-03-fix-jwt-cross-tenant-login-implementation-plan.md` · `2026-06-03-feat-invoice-pdf-jp-font-implementation-plan.md` · `2026-06-03-refactor-approval-merge-service-implementation-plan.md`

```
# TASK: {short imperative title}
_Created: {YYYY-MM-DD} · Status: planning_

## Request
{the user's request, normalized}

## Request analysis
- Intent: {fix-bug | feature | question | refactor}
- Modules involved: {list with numbers}

## Current flow
{Description of the flow based on code you read.
Each step references a concrete file path: `path/file.ext::method() :line`}

## Findings / Root cause / Gap
- Fix-bug: specific root cause, at which file/line
- Feature: gap between current and desired state, files that will be touched
- Question: explanation
- Refactor: current smell + proposed new shape

## Coverage (proof this is complete)
- Symbols traced → where each is used: {symbol → call sites, or "no other callers (grepped: `pattern`)"}
- Hidden-dispatch swept: {events / jobs / observers / DI / cron checked → found … or none}
- Files actually read (not inferred): {list}
- Unverified assumptions: {list, or "none"}

## Plan — checklist (this is the contract; I will not silently deviate)
| # | Action (file → change) | How it will be verified (test / command / manual check) | Status |
|---|---|---|---|
| 1 | {concrete step, file that will change} | {specific test name, command, or manual check that proves it} | ☐ todo |
| 2 | ... | ... | ☐ todo |

_Status legend: ☐ todo · ◐ doing · ✅ done (+evidence) · ⚠️ done-unverified · ❌ blocked._

- Risk / cross-module impact: {what might break in other modules}

## Test plan (decided now, not after the fact)
- Existing tests to run: {commands}
- New test / check needed per behavioral change: {item → test}
- What will be verified manually, and how: {…}

## Definition of done
Every checklist row ✅ with evidence · every changed symbol's call sites re-checked · every behavioral change has a passing test or an explicit "untested because …". Nothing is "done" until this holds.

## Closing reconciliation _(left empty until Step 8)_
- {filled at the end: per-row evidence · what was NOT tested + residual risk · docs updated · pending items}

## Confirm before proceeding?
```

**Hard stop.** Do not touch code until the user reviews this plan and says go. This gate exists to catch a wrong assumption or a missing item *before* implementation, when it's cheap to fix. (In a harness with a plan mode, present the plan there.) For pure Q&A (intent = question), skip the plan-file and gate — but still run the Step 4 trace first for any flow/usage question, then answer directly.

### Step 6 — Execute (work the checklist, don't wander)

After the user confirms:

1. **Track the checklist live — in the task file *and* the harness to-do.** As each row progresses, update its Status in `{tasks_path}/{task-file}` (☐ → ◐ → ✅) and flip the file's `Status: planning` to `in-progress`. The file is the durable copy: if context is lost or the session restarts, re-read it to know exactly where you stopped. This is the cure for "agreed the plan, then forgot half of it."
2. Apply changes **one checklist row at a time**. Do **not** silently add, drop, or reorder scope. If reality forces the plan to change, stop and say so — don't quietly improvise.
3. Run tests if a test suite exists (detect from `composer.json` / `package.json` scripts, `pytest.ini`, `Cargo.toml`, etc.). Tests passing is **necessary, not sufficient** — a green suite over untested hidden paths proves nothing about them
4. Re-grep every symbol you changed and confirm **each call site from the Completeness protocol** is updated or still compatible — including the hidden-dispatch consumers (listeners, jobs, observers)
5. Cross-check: if you modified an endpoint or schema that has cross-references in the docs (`Related: [XX-...]`), verify the related modules don't break

### Step 7 — Update the module docs (REQUIRED if code changed OR insight captured)

After the change is applied, update `{modules_path}/{NN}-{slug}.md` for any of these:

| Change type | Section to update |
|---|---|
| Added/removed file (controller, service, model, FE component, route, middleware, form request) | BACKEND FILES / FRONTEND FILES |
| Changed API endpoint (path, method, payload shape, response shape) | API ENDPOINTS |
| Changed DB schema (new migration, column add/drop, enum change) | DATABASE |
| Changed main workflow / business rule | WORKFLOW / KEY FEATURES |
| Added/removed validation rule | (mention under API ENDPOINTS or NOTES) |
| Discovered docs drift from code (Step 4) | Fix in-place — no separate PR needed |
| **Non-obvious insight discovered during task** (bug root cause, hidden business rule, footgun, undocumented invariant) | KEY FEATURES / WORKFLOW (rules) or NOTES / GOTCHAS (traps) |
| **Module added / removed / renamed**, or a cross-reference (`Related:`) changed | **`00-index.md`** — MODULE LIST + RELATED links (and SYSTEM SCALE if the size class shifts) |
| Domain keywords / DB tables that `00-index.md` lists for a module changed | **`00-index.md`** — that module's entry |

**The 3-question filter for insight capture** — only capture if ALL three are Yes:

1. Is this **not already** in the docs?
2. Would it **surprise** a careful reader of the code? (i.e., not obvious from reading the file directly)
3. Would knowing it save the next reader **>5 minutes**?

Auto-capture vs ask:
- **Bug fix with non-obvious root cause** → default capture (no need to ask; usually passes the filter)
- **Q&A insight** (intent = question, no code change) → ask the user once: *"Did we discover anything worth noting in the docs?"* Default No — keep the bar high.
- **Routine work** (typo, format, trivial rename) → do NOT capture insights, only the file/endpoint change itself.

Rules for updating:

- Edit in-place, preserve the existing format (headings, tables)
- At the end of the file: refresh the footer marker in place — `_Last updated: {today} · Synced: {sync-spec} · Format: v1_`. Rebuild `{sync-spec}` by **discovery**: for the files this module references, group them by owning git repo (walk up to the nearest `.git`) and emit one `{repo-rel-path}@{short SHA}` per repo (non-git files → `{group}#{hash}`; nothing under VCS → `n/a`). This adapts to single-repo, split BE/FE, or monorepo automatically, and is what makes the next session's staleness check (Step 4) work — keep it current.
- **Update `00-index.md` too** when the change affects what it records (module set, `Related:` cross-refs, or the qualitative scale class). 00-index is the entry point `do-task` reads first (Step 2); if it's stale, every future task starts navigating from a wrong map.
- If the task touches multiple modules, update all relevant module docs
- Don't update docs from memory — verify against the code you just changed
- **Docs are reference, not journal.** No per-entry date stamps inside sections. No "LEARNINGS LOG" / changelog accumulation. Each line in the docs must be worth keeping 5 years from now — history lives in `git log`.
- Place insights into the **right existing section**, not a new bucket: business rules → KEY FEATURES/WORKFLOW; traps/footguns → NOTES/GOTCHAS (1 concise line, focused).

### Step 8 — Closing verification gate + report back

**Before you say "done", reconcile the Step 5 checklist row by row — in the task file and in chat** — no exceptions, no "I think it's fine". Fill the file's `## Closing reconciliation` section, update every row's Status, and set the file's `Status:` to `done` (all rows ✅) or `blocked` (any ⚠️/❌). For each row, state one of:
- ✅ done + **evidence** (test name & result, command output, or the manual check you ran)
- ⚠️ done but unverified — and exactly why
- ❌ not done — and why

Then report to the user:
- What you did (files modified)
- **Checklist reconciliation** (the rows above) + **test results** for each behavioral change
- **What was NOT tested, and the residual risk** — state it plainly; never let a gap pass silently
- Which docs you updated
- **Where the task file is**: `{tasks_path}/{task-file}`, now finalized — it's yours to move, archive, or delete
- Any pending items (e.g., migration needs deployment, FE coordination required)

If any row is ⚠️ or ❌, the task is **not** complete — say so up front, don't bury it. "Sorry, I forgot X" after the fact is the exact failure this gate exists to prevent: surface X here, before claiming done.

## Design rationale

**Why docs-first?** The docs have already indexed the codebase and captured business rules. Reading docs first tells you which files are relevant instead of grepping blindly. Saves significant context window on large codebases.

**Why report before acting?** Wrong intent assumption is cheap to fix early; expensive after you've coded. The report also lets the user learn the flow if their goal is understanding.

**Why a checklist + closing gate, not just a plan?** A prose plan is easy to agree on and easy to forget — items silently drop between "sounds good" and "done", and testing gaps surface only when someone asks. The fix isn't "try harder", it's a **contract tracked to the end**: a checklist with a verification per row (Step 5), kept live during execution (Step 6), and reconciled row-by-row with evidence before any "done" claim (Step 8). Forgetting becomes structurally visible instead of an after-the-fact apology.

**Why is updating docs mandatory?** This skill works because docs are accurate. Every code change without a doc update = docs rot. After N rots, the skill becomes useless and a full re-audit is required (expensive). Updating now is the cheapest insurance.

**Why not grep directly?** Wrong framing — grep is not optional and not a backup. Docs tell you *where to start*; grep tells you *where the flow ends*. The win from docs isn't "grep less", it's "grep targeted instead of blind": you learn which symbols to chase, then you still chase every one of them across the whole repo (Completeness protocol). On a large codebase with hidden flows, skipping the usage grep is exactly how you ship a confident, wrong change. Map first, then walk the whole territory — never confuse the map for the territory.

**Why no config file?** A conventional docs path (`docs/modules/` or `<repo>-docs/modules/`) is enough — auto-discovery handles 95%+ of cases. Locale and project roots are inferable from the docs themselves (file paths inside docs are already relative to project root). A config file would be another thing to maintain for negligible gain.

**Why no LEARNINGS LOG / changelog inside docs?** Docs are a **reference**, not a journal. Append-only logs with per-entry dates bloat the file, push the authoritative content down, and after a few months the doc reads like a git log instead of a spec — at which point nobody reads it and the whole system collapses. Insights worth keeping go into the right semantic section (KEY FEATURES, NOTES/GOTCHAS) with no timestamp; history lives in `git log` and is queryable when needed. The 3-question filter in Step 7 keeps capture-rate low on purpose — high signal beats high volume.

## Anti-patterns to avoid

- ❌ Skipping Step 0 (locate folder) → reading the wrong path → fail
- ❌ Skipping 00-index → wandering into the wrong module
- ❌ Reading the whole module folder instead of the relevant files → wastes context
- ❌ Acting without reporting → user loses the chance to correct early
- ❌ Skipping doc updates because "lazy" → creates debt for the next session
- ❌ Updating docs from memory rather than verifying against code → wrong docs
- ❌ Trusting docs blindly when code has clearly changed → applying outdated business rules
- ❌ Reading docs but not code → answering from stale information
- ❌ Ad-hoc indexing instead of calling `index-codebase` → non-standard format that breaks future use
- ❌ Capturing every insight / writing dated entries / adding a "LEARNINGS LOG" → docs bloat into a journal, lose trust, get ignored
- ❌ Capturing insights without the 3-question filter → low-signal noise dilutes the high-signal content already in the docs
- ❌ Concluding a root cause / behavior from the docs or a function name without reading the code → confidently wrong
- ❌ Reading only the module's own files and assuming the flow is contained there → misses cross-module and hidden-dispatch callers
- ❌ Changing a symbol without grepping the whole repo for its usages → silent breakage at an unseen call site
- ❌ Claiming "no other callers / nothing else affected" without having grepped → absence of evidence ≠ evidence of absence
- ❌ Skipping the hidden-dispatch sweep (events, jobs, observers, DI, cron) → the real driver of the flow goes unseen
- ❌ Treating a behavioral change as "trivial" to skip the Completeness protocol → that's exactly how hidden flows break
- ❌ Presenting a plan, then quietly implementing something different or dropping items → the agreed checklist is the contract, tracked to the end
- ❌ Claiming "done" without walking the Step 5 checklist row by row → that's how items get forgotten
- ❌ Skipping/hand-waving tests, or hiding what wasn't tested → "tests full of gaps" then "sorry I forgot" is exactly what the closing gate forbids

## Output style

- Output to the user matches the user's working language (the language of their most recent message).
- Mixing English for technical terms is fine.
- Concise — bullets and tables when listing many items.
- File paths **relative to project root**.
- Code reference format: `path/to/file.ext::methodName()` or `:line-N`.
- When listing many files, group by type (Controllers / Services / Models / Routes / FE).

---
_Last updated: 2026-06-03_
