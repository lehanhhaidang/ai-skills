---
name: task-workflow
description: Disciplined workflow for handling coding tasks with reference documents. Use this skill whenever the user provides a task along with a path to a reference document (spec, design doc, ticket, requirements file), or whenever they ask you to implement, modify, fix, or extend something based on documentation. Trigger this even if they don't explicitly say "follow my workflow" — any combination of "here's the doc + here's the task" should activate it. Covers reading the document, reconciling it against current code (code is source of truth), proposing a plan when changes are non-trivial, getting approval, implementing, self-checking, and reporting back. Works across Claude Code, Antigravity, Codex, terminal CLIs, and other AI coding agents — adapts to whichever plan mode the platform offers.
---

# Task Workflow

A structured workflow for tackling coding tasks where the user supplies a reference document and expects careful, predictable execution. The goal: no surprises, no silent assumptions, no broken hidden flows.

## Core principle

**Code is the source of truth.** Documents describe intent; code describes reality. When they disagree, trust the code and flag the divergence to the user.

**Always confirm before writing code.** Even tiny edits get a heads-up first. The user explicitly wants to be in the loop on every change, regardless of size.

## The 6 phases

Walk through these in order. Don't skip phases — but do compress them when context allows (e.g., if the user already explained the task in detail, Phase 1 might just be a one-line restatement).

### Phase 1 — Receive task + document

The user gives you:
- A task description (may be clear, may be vague)
- A path to a reference document

Acknowledge briefly. If the task is unclear, ask now — not after reading the doc. Don't pretend to understand something you don't.

### Phase 2 — Read the document, then reconcile with code

1. **Read the document fully** at the given path.
2. **Identify any code references** in the document — file paths, function names, snippets, API contracts, schemas, configs.
3. **Open the actual files** referenced and compare against what the document claims.
4. **Report divergences** to the user before proceeding. Examples:
   - "The doc references `getUserById()` but the current code has `findUserById()`."
   - "Doc says response shape is `{data, error}` but code returns `{result, err}`."
   - "Doc mentions a `users` table but the schema shows `user_accounts`."

When the doc and code disagree, **the code wins**. State this clearly: "I'll treat the code as source of truth and adjust accordingly."

If the doc and code fully match, just say so in one line and move on.

### Phase 3 — Scope the work

After understanding the task and the current code, decide the scope:

- **How many files will change?**
- **Are there hidden flows that could break?** (event handlers, side effects, shared state, config-driven branches, downstream consumers, scheduled jobs, migrations)
- **Is this trivial or non-trivial?**

**Threshold**: If the change touches **3 or more files**, OR involves significant logic changes, OR affects shared/exported APIs — it's non-trivial. Go to Phase 4 (planning).

If it's truly small (1-2 files, isolated change, no API surface impact) — still do not silently start coding. Briefly describe what you intend to change and ask the user to confirm before touching code. The user has explicitly asked for a confirmation step on every task regardless of size.

### Phase 4 — Plan (when required)

If the active platform has a **plan mode** (Claude Code plan mode, Antigravity plan, Codex planning, etc.), use it. The platform will enforce the plan-then-execute split. If it doesn't, present the plan inline as a regular message and wait for explicit approval before implementing.

The plan should cover these sections (adapt naming/wording to fit the situation; this is a checklist of what the user wants visible, not a rigid template):

1. **Original requirement / problem** — restate what the user is asking for, in your own words.
2. **Current state of the code** — what exists today that's relevant.
3. **What's missing** — the gap between current state and the goal.
4. **Implementation approach** — the plan for how to close the gap.
5. **Affected files** — concrete list, with a one-line note on what changes in each.
6. **Post-implementation state** — what will be true after the change is done.
7. **Risks / hidden flows** — what could break, what side effects exist, what the user should be aware of. **You're responsible for avoiding these breakages during implementation; this section just makes them visible so the user can sleep at night.**

Wait for explicit approval ("ok", "go ahead", "approved", etc.) before moving to Phase 5. If the user pushes back or asks questions, refine the plan and re-confirm.

### Phase 5 — Implement

Once approved:

1. Make the changes per the plan.
2. Stay within scope — don't drift into adjacent refactors unless the user agreed.
3. If you discover something mid-implementation that invalidates the plan (e.g., a hidden dependency you missed), stop, report, and re-plan rather than improvising.
4. After writing the changes, **self-check**:
   - Run the project's build/typecheck/test/lint as appropriate. Detect what's available — `package.json` scripts, `Makefile` targets, `pyproject.toml`, `go.mod`, etc.
   - If the project has a test runner and tests exist for the touched areas, run them.
   - If the build/test fails, **read the error and fix it yourself** before reporting. Don't hand the user a broken state.
   - If you can't run the build (no permissions, missing env, etc.), say so explicitly.

#### Choosing what to run for self-check

| Stack | First things to try |
|---|---|
| JS/TS/React/Next | `npm run build`, `npm run typecheck`, `npm run lint`, `npm test` (whichever exist in `package.json`) |
| Python | `python -m pytest`, `mypy`, `ruff`, `python -c "import <module>"` for smoke import |
| Go | `go build ./...`, `go vet ./...`, `go test ./...` |
| Node backend | same as JS/TS plus any `npm run start:dev` smoke if applicable |
| Other | look for `Makefile`, `justfile`, `taskfile.yml`, CI config, README |

If the project has no build/test infra, do a manual code-trace: re-read the changed files, verify imports resolve, check that you didn't leave broken syntax or unfinished blocks.

### Phase 6 — Report back

Final message to the user, structured as:

- **What I did** — short summary of the change.
- **Problem solved** — tie back to the original requirement.
- **Files modified** — concrete list.
- **Self-check result** — what you ran, what passed/failed, anything skipped and why.
- **Compatibility note** — explicitly state whether existing logic was preserved or whether anything intentionally changed behavior.

Keep it tight. The user wants to confirm the work, not read a novel.

## Platform adaptation

You may run inside Claude Code, Antigravity IDE, Codex CLI/desktop, or other agent environments. The 6 phases stay the same; only the mechanism for "planning" varies:

- **Plan mode available** (e.g., Claude Code plan mode, Antigravity plan): use it. The platform separates plan from execution natively.
- **No plan mode**: present the plan as a normal chat message and explicitly say "Waiting for your approval before I make any changes." Do not start editing files until the user replies affirmatively.
- **Auto-execute / YOLO mode**: even here, post the plan first and pause. The user has been clear that they want a confirmation step regardless of platform defaults.

If you're unsure whether plan mode is active, just say so and present the plan inline — it's always safe to over-communicate.

## Quick reference

```
Task + doc path
    ↓
Read doc → cross-check with code → report divergences
    ↓
Trivial? → describe change, ask to confirm, then implement
Non-trivial? → present plan with the 7 sections, wait for approval
    ↓
Implement within scope
    ↓
Run build/test/lint, fix anything that fails
    ↓
Report: what / why / files / check / compat
```

## Anti-patterns to avoid

- **Trusting the doc over the code.** Docs go stale. Code runs.
- **Silently expanding scope.** "I also refactored X while I was there" — don't, unless asked.
- **Skipping the self-check** because the change "looked fine."
- **Hiding broken builds.** If the typecheck fails after your edit, fix it or report it. Don't leave the user to discover it.
- **Surprise-edits on small tasks.** The user has explicitly asked for confirmation on every task, no matter the size.
- **Restating risks without avoiding them.** The risks/hidden-flows section is for visibility — you still need to actually handle the breakages, not just list them.
