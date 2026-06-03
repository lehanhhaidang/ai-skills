---
name: index-codebase
description: Use this skill when starting work on an unfamiliar codebase that needs a navigation map, when the user asks to "index this project", "generate module docs", "build a code map", "document the architecture", "scan the repo and document modules", or when the `do-task` skill complains "no modules folder found / docs not initialized". This skill walks the entire repository, identifies business modules (by domain when possible, by technical layer as fallback), and generates a `modules/` folder (placed under `docs/` or a project-named docs directory) containing `00-index.md` + one Markdown doc per module. Run this ONCE per new project; subsequent updates are incremental via `do-task` itself.
---

# Index Codebase — Generate Module Documentation Map

## Purpose

Generate a **navigation map** for any codebase (any language / framework): a `modules/` folder containing `00-index.md` plus N module docs, each describing one business module.

This map is the input the `do-task` skill consumes. Without it, `do-task` would have to rediscover the codebase on every invocation — wasteful and error-prone.

## When to trigger

- A project is being opened for the first time and has no `modules/` docs
- User requests "create docs for this project", "index the codebase", "document this project"
- The `do-task` skill reports it cannot find `00-index.md`
- The codebase recently underwent a large change (rewrite, big branch merge) and the user wants a re-index

## Philosophy

- **Standard format**: docs must follow a consistent shape so other skills can parse them automatically
- **Domain-first grouping**: split modules by business domain (auth, billing, vehicle…), not by technical layer (controllers, services, models) — unless the codebase is small or pure infrastructure
- **Parallel deep-dive**: one sub-agent per module reads code deeply — grep-verified, never skim-and-guess; outputs are uniform because they share the same template
- **Confirm before generating**: outline the module list, get user confirmation, then run the expensive agent passes

## Workflow

### Step 1 — Detect project structure

Walk top-level directories. Identify:

- **Languages / frameworks**:
  - Backend: `composer.json` (PHP/Laravel), `package.json` + `next.config*` (Next.js), `pom.xml` (Java/Spring), `go.mod`, `Cargo.toml`, `requirements.txt` / `pyproject.toml`, `Gemfile`, `mix.exs`, etc.
  - Frontend: `package.json` with React/Vue/Angular/Svelte, or Blade/EJS/Pug templates
- **Repo shape**: single repo · monorepo (one repo, many apps) · **multi-repo** (separate BE/FE or service repos under one parent — check sub-folders for their own `.git`). Note backend root, frontend root, docs root, and which roots are distinct git repos (the `Synced:` footer depends on this).
- **Existing docs**: `README.md`, `CHANGES*.md`, `docs/`, `<repo>-docs/`, OpenAPI specs — anything reusable.
- **Directory layout**: controllers/, services/, models/, routes/, components/, pages/, lib/, etc.

Give the user a one-line summary: "I see this is X (BE: Laravel, FE: Next.js), monorepo with two roots `api/` and `web/`. Correct?" (name the actual versions you detect from the manifests — don't hardcode any.)

### Step 2 — Interview the user (concise)

Ask 2–4 questions, only the ones that matter:

1. **Docs output path**: default `docs/modules/`, or a project-named convention like `<repo>-docs/modules/`?
2. **Grouping strategy**: by domain (preferred), by layer, or does the user have a specific idea?
3. **Docs language**: English / Vietnamese / mix? (Defaults to the user's working language.)
4. **Scope**: full deep-dive per module, or quick outline-only?

Defaults should be sensible — only override when the user explicitly cares.

### Step 3 — Outline the module list (cheap pass)

Skim just enough to list modules — do **not** deep-read files yet:

- List controllers + routes → infer domains (e.g., AuthController + UserController → "Authentication", "User Management")
- List DB tables → group by prefix or relationship
- List FE pages → confirm groupings from the frontend side

Draft `00-index.md` containing:

- Tech stack overview
- Module list (NN + name + 1-sentence description + key files placeholder)

Show the draft to the user. Wait for confirmation or edits. **This is the cheapest place to fix mistakes — before spending agent runs.**

### Step 4 — Generate per-module docs (parallel agents)

After the outline is confirmed, spawn **one sub-agent per module** (in parallel if the environment allows; sequentially if the user prefers tight quality control).

**If the environment has no sub-agent capability at all** (can't spawn isolated contexts): do NOT deep-read every module in one context — that's exactly the 1.6M-token blow-up the per-agent design exists to avoid. Degrade to **incremental indexing across turns**: index a few modules per turn, write each `{NN}-{slug}.md` to disk as you finish it, then drop those files from context before the next batch; finalize `00-index.md` last. A coarser fallback is an **outline-only first pass** (OVERVIEW + KEY FEATURES + file lists; skip deep WORKFLOW / TRIGGERS), then deepen each module on demand the first time `do-task` touches it.

Each agent receives:

- Module number + name + 1-sentence intent
- Code roots (BE/FE paths)
- The format template (see "Format Spec" below)
- A map of each code root → its short SHA (`git rev-parse --short HEAD` per repo) plus the format version (`v1`), so each agent stamps the `Synced:` footer with only the repos *its* module actually touches; agents may not have git access of their own
- Instruction: read code, output one complete file `modules/{NN}-{slug}.md`

Each agent's tasks:

1. List relevant controllers, read method signatures
2. List services + main business logic
3. List models + DB tables (read migrations)
4. List routes + API endpoints (method/path/auth/payload)
5. List form requests / validation
6. List relevant middleware
7. List FE pages + components
8. Trace the main workflow (if one exists)
9. Note edge cases and business rules that aren't obvious from code
10. **Trace inbound triggers** — who drives this module from outside: events it consumes, queued jobs, scheduled tasks, observers, and calls from other modules. Grep across the repo; don't guess from filenames.
11. **Trace outbound side effects** — what this module sets off elsewhere: events emitted, jobs dispatched, notifications sent, calls into other modules. These hidden flows are what `do-task` most needs on the map.

### Step 5 — Finalize 00-index.md

After all module agents finish:

- Gauge scale qualitatively (small / medium / large per dimension) — do NOT bake exact counts of controllers/services/models into the docs; precise counts rot on the first commit and add no navigational value
- Update 00-index with:
  - Module summaries (pulled from KEY FEATURES + main API ENDPOINTS of each module)
  - Architecture overview (databases, portals if applicable)
  - System scale (qualitative buckets — see SYSTEM SCALE; not exact counts)
  - Cross-references between modules

### Step 6 — Report

Report to the user:

- N modules generated, path `{modules_path}/`
- Pick a path `do-task` can auto-discover: `docs/modules/`, `<repo>-docs/modules/`, or `modules/`. Other paths will work too but require the user to point `do-task` at them manually.
- Next step: use the `do-task` skill for the next coding task

## Format Spec

This skill MUST emit docs in the format below so `do-task` can read them automatically.

**Footer marker (every generated file).** Each doc ends with one reference line — a field, not a journal:
`_Last updated: {YYYY-MM-DD} · Synced: {sync-spec} · Format: v1_`

- `Synced: {sync-spec}` = the VCS baseline at which this doc was verified, **discovered from the files the module references — never assumed to be a single repo.** Build it at write/verify time:
  1. For each file path the doc lists, find its owning repo by walking up to the nearest `.git`.
  2. Group files by repo. Emit one entry per distinct repo: `{repo-path-relative-to-project-root}@{short SHA}` (`git rev-parse --short HEAD` in that repo).
  3. Files under no git repo → `{group}#{short content hash}`; if there's no VCS at all → `n/a`.
  4. Join entries with ` · `.

  This adapts to any layout automatically: single repo → `app@a1b2c3` · split BE/FE → `drivercall-be@a1b2c3 · drivercall-fe@d4e5f6` · monorepo → one entry · no VCS → `n/a`. `do-task` reads it back and re-checks each entry to detect drift.
- `Format: v1` = the version of this format spec, so `do-task` knows whether the docs match what it expects. Bump only when the spec's shape changes.
- Updated **in place**, never appended to — stays compatible with the "docs are reference, not journal" rule.

### File `{modules_path}/00-index.md`

```markdown
# {SYSTEM NAME} - MODULE INDEX

## 1. PROJECT INTRODUCTION

**{Project Name}** is {short description}.

### Main technologies
- **Backend**: {framework, version, key libraries}
- **Frontend**: {framework, version, key libraries}
- **Database**: {primary DB(s), any specialized stores}
- **Deployment**: {Docker / Kubernetes / etc., if known}

### Production URLs (if known)
- **Frontend**: {url or "N/A"}
- **Backend API**: {url or "N/A"}

---

## 2. MODULE LIST

### 01. {Module Name}

**Description**: {2-3 sentence summary}

**Key Features**:
- {bullet 1}
- {bullet 2}

**Main API Endpoints**:
| Method | Endpoint | Description |
|---|---|---|
| ... | ... | ... |

**Frontend Pages**:
- `{path}` - {description}

**Database**: `{table1}`, `{table2}`, ...

**Related**: [{NN}-{slug}](./{NN}-{slug}.md), ...

---

### 02. {Next Module}
...

---

## 3. SYSTEM ARCHITECTURE

### Databases
| Database | Purpose | Tables |
|---|---|---|
| ... | ... | ... |

### Portals / Entry points (if applicable)
| Portal | URL Prefix | Users |
|---|---|---|
| ... | ... | ... |

---

## 4. SYSTEM SCALE

_Qualitative size, not a live tally — exact counts rot on the first commit and nobody updates them. Use order-of-magnitude buckets that survive normal churn._

| Dimension | Scale |
|---|---|
| Backend surface (controllers / services) | {small / medium / large} |
| Data model (tables / models) | {small / medium / large} |
| Frontend (screens / components) | {small / medium / large} |
| Module docs | {N} files |

Buckets (rough): **small** ≈ <10 · **medium** ≈ 10–40 · **large** ≈ 40+ per dimension. `Module docs: {N}` is fine to keep exact — it changes rarely.

---
_Last updated: {YYYY-MM-DD} · Synced: {sync-spec} · Format: v1_
```

### File `{modules_path}/{NN}-{slug}.md`

```markdown
# MODULE: {MODULE NAME IN CAPS}

## OVERVIEW
{1-3 sentences describing what this module does at a high level}

---

## KEY FEATURES
- **{Feature 1}**: {detail}
- **{Feature 2}**: {detail}
- ...

---

## BACKEND FILES

### Controllers
| File | Description |
|---|---|
| `{relative/path/Controller.ext}` | {responsibility} |

### Models
| File | Table | Description |
|---|---|---|
| `{path}` | `{table_name}` | {what it represents} |

### Services
| File | Description |
|---|---|

### Routes
| File | Prefix |
|---|---|

### Form Requests / Validation
| File | Description |
|---|---|

### Middleware (if applicable)
| File | Description |
|---|---|

### Config / Constants (if applicable)
| File | Description |
|---|---|

---

## API ENDPOINTS

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/path` | guard | what it returns |
| POST | `/path` | guard | what it does |

---

## FRONTEND FILES

### Pages
- `{path}` — {description}

### Components
- `{path}` — {description}

---

## DATABASE

### Tables
| Table | Purpose |
|---|---|

### Schema highlights
- `{table}.{column}` — type, constraints, semantic notes
- Enum values worth knowing: ...

---

## WORKFLOW (if a clear flow exists)

{Step-by-step description, or state machine description}

---

## TRIGGERS & SIDE EFFECTS (hidden flows)

_What drives this module and what it sets off — the indirection that grep, not reading the controller, reveals. Grep-verified, not guessed. Leave a section empty only if you searched and found none._

### Inbound (what invokes this module)
- {event consumed / job / schedule / observer / external caller} → handled in `{path}::{method}()`

### Outbound (what this module sets off)
- {event emitted / job dispatched / notification / cross-module call} → from `{path}::{method}()`, handled in [{NN}-{slug}]

---

## NOTES / GOTCHAS

- {Edge case 1}
- {Business rule not obvious from code}
- {Deadcode flag: ...}
- {Notable TODO/FIXME: ...}

---

## RELATED MODULES

- [{NN}-{slug}](./{NN}-{slug}.md) — {why related}

---
_Last updated: {YYYY-MM-DD} · Synced: {sync-spec} · Format: v1_
```

## Decisions when the codebase doesn't match perfectly

**Very small codebase** (< 5 controllers, no clear domain): output 3-5 modules grouped by layer (api / data / ui / infra) rather than forcing domain grouping.

**Very large codebase** (50+ potential modules): consolidate sensibly down to ~16-20 modules. The ceiling isn't arbitrary — the index has to be readable in one pass and a person has to hold the module list in their head; past ~20, nobody can navigate it and it stops being a map.

**No existing docs**: proceed normally — this is greenfield indexing.

**High-quality existing docs**: skim them, merge any still-valuable notes, but verify everything against code (docs may have rotted).

**Microservices repo**: each service may be its own "system" — confirm with the user whether to index each service separately or treat the whole repo as one system.

**Large monorepo (Nx/Turborepo/Bazel)**: ask about scope. Indexing per-app under `apps/*/docs/modules/` is often the right shape.

**Multi-repo project (separate BE/FE or service repos under one non-git parent)**: common and fully supported. Treat the parent folder as the project root and each sub-folder as its own repo; a single module doc may legitimately span several repos. The `Synced:` footer records one `repo@sha` entry per repo the module touches (see Format Spec) — discovered, not hardcoded. Note: if the docs folder itself sits outside every git repo, its history won't be in any `git log` — suggest the user keep docs in a tracked location if they want that safety net.

## Anti-patterns

- ❌ Generating full docs before confirming the module list with the user → wasted agent runs
- ❌ Reading one file and guessing at function behavior → wrong docs (always verify by code)
- ❌ Too few modules (1-2) or too many (50+) → useless map
- ❌ Grouping by technical layer when domain grouping is clearly possible → docs don't reflect business
- ❌ Hardcoding paths from one project's convention → forgetting to configure for the actual project
- ❌ Skipping the inbound/outbound trigger sweep → hidden flows stay invisible, and `do-task` later reads a map that hides exactly the couplings that cause bugs

## Design rationale

**Why a standard format?** Not because anything literally parses the Markdown — `do-task` is a model reading prose, not a JSON parser. The value is **orientation**: identical section names across every module (BACKEND FILES, API ENDPOINTS, TRIGGERS…) mean the model knows exactly where to look without re-learning each project's ad-hoc layout. Consistency is what makes the docs fast and reliable to consume; bespoke per-project formats force a fresh read every time and defeat the point.

**Why parallel agents?** Each module's deep-dive consumes 100k-200k tokens. Doing 16 modules in **one** conversation context = 1.6M+ tokens → blows up. Parallel sub-agents each get their own context. Sequential sub-agents are equally fine for context (each still isolated) — they just trade wall-clock for tighter control. The only real failure mode is an environment with **no** sub-agents at all; Step 4's incremental-indexing fallback covers that.

**Why confirm the outline first?** Wrong grouping at outline = cheap to fix (edit text). Wrong grouping after full generation = re-spawning agents = expensive.

**Why no config file?** The output path is the only piece `do-task` needs, and placing docs at a conventional location (`docs/modules/` or `<repo>-docs/modules/`) lets `do-task` find them with no setup. A config file would be one more thing to maintain for negligible gain.

## Output style

- The DOCS produced (Markdown content) follow the user's chosen locale (Step 2). Module section headers (`OVERVIEW`, `KEY FEATURES`, etc.) stay in English for portability; prose inside can be in any language.
- File paths in docs: **relative to project root**.
- Code reference format: `path/to/file.ext::symbol()` or `:line-N`.
- Concise — prefer bullets and tables when listing many items.

---
_Last updated: 2026-06-03_
