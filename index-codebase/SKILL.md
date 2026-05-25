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
- **Parallel deep-dive**: one sub-agent per module reads code deeply; outputs are uniform because they share the same template
- **Confirm before generating**: outline the module list, get user confirmation, then run the expensive agent passes

## Workflow

### Step 1 — Detect project structure

Walk top-level directories. Identify:

- **Languages / frameworks**:
  - Backend: `composer.json` (PHP/Laravel), `package.json` + `next.config*` (Next.js), `pom.xml` (Java/Spring), `go.mod`, `Cargo.toml`, `requirements.txt` / `pyproject.toml`, `Gemfile`, `mix.exs`, etc.
  - Frontend: `package.json` with React/Vue/Angular/Svelte, or Blade/EJS/Pug templates
- **Repo shape**: monorepo (multiple roots) vs single. Note backend root, frontend root, docs root if any.
- **Existing docs**: `README.md`, `CHANGES*.md`, `docs/`, `<repo>-docs/`, OpenAPI specs — anything reusable.
- **Directory layout**: controllers/, services/, models/, routes/, components/, pages/, lib/, etc.

Give the user a one-line summary: "I see this is X (BE: Laravel 12, FE: Next.js 15), monorepo with two roots `api/` and `web/`. Correct?"

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

Each agent receives:

- Module number + name + 1-sentence intent
- Code roots (BE/FE paths)
- The format template (see "Format Spec" below)
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

### Step 5 — Finalize 00-index.md

After all module agents finish:

- Aggregate stats: count controllers, services, models, routes, FE screens, migrations, components
- Update 00-index with:
  - Module summaries (pulled from KEY FEATURES + main API ENDPOINTS of each module)
  - Architecture overview (databases, portals if applicable)
  - Stats table
  - Cross-references between modules

### Step 6 — Report

Report to the user:

- N modules generated, path `{modules_path}/`
- Pick a path `do-task` can auto-discover: `docs/modules/`, `<repo>-docs/modules/`, or `modules/`. Other paths will work too but require the user to point `do-task` at them manually.
- Next step: use the `do-task` skill for the next coding task

## Format Spec

This skill MUST emit docs in the format below so `do-task` can read them automatically.

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

## 4. SYSTEM-WIDE STATS

| Component | Count |
|---|---|
| Controllers | {N} |
| Services | {N} |
| Models | {N} |
| Middleware | {N} |
| Form Requests | {N} |
| Routes | {N} |
| Migrations | {N} |
| Frontend screens | {N} |
| Shared components | {N} |
| API endpoints | {N} |
| Module docs | {N} files |

---
_Last updated: {YYYY-MM-DD}_
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

## FRONTEND PAGES & COMPONENTS

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

## NOTES / GOTCHAS

- {Edge case 1}
- {Business rule not obvious from code}
- {Deadcode flag: ...}
- {Notable TODO/FIXME: ...}

---

## RELATED MODULES

- [{NN}-{slug}](./{NN}-{slug}.md) — {why related}

---
_Last updated: {YYYY-MM-DD}_
```

## Decisions when the codebase doesn't match perfectly

**Very small codebase** (< 5 controllers, no clear domain): output 3-5 modules grouped by layer (api / data / ui / infra) rather than forcing domain grouping.

**Very large codebase** (50+ potential modules): consolidate sensibly down to ~16-20 modules. Too many modules = unusable map.

**No existing docs**: proceed normally — this is greenfield indexing.

**High-quality existing docs**: skim them, merge any still-valuable notes, but verify everything against code (docs may have rotted).

**Microservices repo**: each service may be its own "system" — confirm with the user whether to index each service separately or treat the whole repo as one system.

**Large monorepo (Nx/Turborepo/Bazel)**: ask about scope. Indexing per-app under `apps/*/docs/modules/` is often the right shape.

## Anti-patterns

- ❌ Generating full docs before confirming the module list with the user → wasted agent runs
- ❌ Reading one file and guessing at function behavior → wrong docs (always verify by code)
- ❌ Too few modules (1-2) or too many (50+) → useless map
- ❌ Grouping by technical layer when domain grouping is clearly possible → docs don't reflect business
- ❌ Hardcoding paths from one project's convention → forgetting to configure for the actual project

## Design rationale

**Why a standard format?** The `do-task` skill reads docs programmatically. A uniform format makes parsing trivial. Per-project ad-hoc formats are useless.

**Why parallel agents?** Each module's deep-dive consumes 100k-200k tokens. Sequentially doing 16 modules = 1.6M+ tokens in one conversation context → blows up. Parallel sub-agents each get their own context.

**Why confirm the outline first?** Wrong grouping at outline = cheap to fix (edit text). Wrong grouping after full generation = re-spawning agents = expensive.

**Why no config file?** The output path is the only piece `do-task` needs, and placing docs at a conventional location (`docs/modules/` or `<repo>-docs/modules/`) lets `do-task` find them with no setup. A config file would be one more thing to maintain for negligible gain.

## Output style

- The DOCS produced (Markdown content) follow the user's chosen locale (Step 2). Module section headers (`OVERVIEW`, `KEY FEATURES`, etc.) stay in English for portability; prose inside can be in any language.
- File paths in docs: **relative to project root**.
- Code reference format: `path/to/file.ext::symbol()` or `:line-N`.
- Concise — prefer bullets and tables when listing many items.

---
_Last updated: 2026-05-21_
