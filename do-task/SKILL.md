---
name: do-task
description: Use this skill whenever working on any codebase that has been indexed with a modules documentation system — when fixing bugs, implementing features, explaining how something works, tracing a flow across files, refactoring, or answering any question about the architecture, business domain, specific files, or "where is X defined / who calls Y". The skill auto-detects the modules folder by checking common paths (`docs/modules/`, `*-docs/modules/`, `modules/`, `documentation/modules/`), reads `00-index.md` to identify relevant modules, then reads module docs + actual code BEFORE acting. This is dramatically faster than grepping cold. After making changes, the skill updates module docs to keep them in sync so they don't rot. If no modules folder is found, the skill asks whether docs live at a non-standard path or offers to invoke `index-codebase` on the spot, then continues the task seamlessly. Use this BEFORE touching code in any indexed project.
---

# Do-Task — Module-First Codebase Navigator (Project-Agnostic)

## Philosophy

**Docs are the map. Code is the territory.** Read the map before moving. Update the map after changing the terrain.

This skill applies to any codebase indexed by `index-codebase` (or an equivalent process): a `modules/` folder with `00-index.md` plus N module docs in a standard format.

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
- **NOTES / GOTCHAS** — edge cases, deadcode flags, business rules that aren't obvious from code

Note which important functions live in which files, and which business rules apply to the current task.

### Step 4 — Read the actual code (selective — not the whole module)

Use the file paths from the docs. **Only read files genuinely relevant to the intent:**

- Bug at endpoint → controller method → service it calls → model (if the task touches schema/data)
- Feature touching UI → FE page + components + the BE endpoint it will call + the backend service
- Question about a workflow → trace from entry point (route/controller) through services, stop when you understand enough
- Refactor → read the "before" shape and verify caller dependencies

While reading, hold on to: what each function does, its inputs/outputs, and side effects (notifications dispatched, recalc triggered, DB writes).

If docs drift from code (e.g., docs mention a method that no longer exists, or code has a method docs missed) → flag it in the report. Don't trust docs blindly. Note these for the doc update in Step 7.

### Step 5 — Analysis report (BEFORE acting)

Return a report to the user following this template (in the user's locale; the headings below are placeholders):

```
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

## Proposed actions
1. {concrete step, file that will change}
2. ...
- Risk / cross-module impact: {what might break in other modules}

## Confirm before proceeding?
```

Wait for the user's OK before modifying code. For pure Q&A (intent = question), you can skip the confirm and answer directly.

### Step 6 — Execute

After the user confirms:

1. Apply changes sequentially according to the plan
2. Run tests if a test suite exists (detect from `composer.json` / `package.json` scripts, `pytest.ini`, `Cargo.toml`, etc.)
3. Cross-check: if you modified an endpoint or schema that has cross-references in the docs (`Related: [XX-...]`), verify the related modules don't break

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
- At the end of the file: replace the old timestamp with `_Last updated: YYYY-MM-DD_`
- If the task touches multiple modules, update all relevant module docs
- Don't update docs from memory — verify against the code you just changed
- **Docs are reference, not journal.** No per-entry date stamps inside sections. No "LEARNINGS LOG" / changelog accumulation. Each line in the docs must be worth keeping 5 years from now — history lives in `git log`.
- Place insights into the **right existing section**, not a new bucket: business rules → KEY FEATURES/WORKFLOW; traps/footguns → NOTES/GOTCHAS (1 concise line, focused).

### Step 8 — Report back

Tell the user briefly:
- What you did (files modified)
- What you tested/verified
- Which docs you updated
- Any pending items (e.g., migration needs deployment, FE coordination required)

## Design rationale

**Why docs-first?** The docs have already indexed the codebase and captured business rules. Reading docs first tells you which files are relevant instead of grepping blindly. Saves significant context window on large codebases.

**Why report before acting?** Wrong intent assumption is cheap to fix early; expensive after you've coded. The report also lets the user learn the flow if their goal is understanding.

**Why is updating docs mandatory?** This skill works because docs are accurate. Every code change without a doc update = docs rot. After N rots, the skill becomes useless and a full re-audit is required (expensive). Updating now is the cheapest insurance.

**Why not grep directly?** Grep is a backup, not the primary tool. Grep is great for "find all usages" once you know the symbol. Docs are great for "where do I start". They complement each other, not substitute.

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

## Output style

- Output to the user matches the user's working language (the language of their most recent message).
- Mixing English for technical terms is fine.
- Concise — bullets and tables when listing many items.
- File paths **relative to project root**.
- Code reference format: `path/to/file.ext::methodName()` or `:line-N`.
- When listing many files, group by type (Controllers / Services / Models / Routes / FE).

---
_Last updated: 2026-05-25_
