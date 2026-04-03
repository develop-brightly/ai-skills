---
name: feature-planner
description: >
  Plan and scope new features on an existing codebase. Use this skill whenever a
  user wants to implement something new, understand what needs to change to add a
  feature, analyze requirements, write a technical spec, figure out what files or
  database tables need to change, or translate business requirements into an
  implementation plan. Trigger on phrases like: "I want to add X", "how would I
  implement Y", "what would it take to build Z", "help me plan this feature",
  "I need to scope out this change", "what files would I need to touch", or any
  time a user describes a product idea or enhancement and wants to understand the
  engineering work involved. When in doubt, use this skill — the structured
  output it produces prevents costly rework from missed edge cases or schema gaps.
---

# Feature Planner

A skill for turning vague feature requirements into clear, actionable implementation plans grounded in the actual codebase.

The core problem this skill solves: requirements stated at a product level ("add social login") leave out dozens of engineering decisions. The goal is to explore the real codebase, fill in those decisions with specifics, surface ambiguities before code gets written, and produce a plan precise enough that any developer on the team could pick it up.

---

## Step 1: Intake requirements

Determine how the user is providing requirements. They may have:
- Described the feature conversationally in this message
- Pasted a spec or ticket inline
- Referenced a file containing requirements

If requirements are vague or high-level — which they usually are — that's fine. Part of this skill's job is to elaborate them. Do not demand a detailed spec before proceeding. Extract whatever intent is present and move to codebase exploration; the exploration will reveal what questions need answering.

If the user referenced a file path, read it now.

---

## Step 2: Explore the codebase

Systematic exploration is what separates a generic plan from a useful one. The goal is to build a mental model of the architecture before making any recommendations. Use parallel subagents where possible to do this quickly.

**Start with orientation.** Read the top-level directory structure, then the manifest file — `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, or equivalent. This reveals the tech stack, framework, and key dependencies in one step. Also check for a `README.md` at the repo root.

**Identify the architectural pattern.** Is this a monolith, microservices, a monorepo? Where is the web layer? Where is business logic? Data access? Look for directories named: `routes/`, `controllers/`, `handlers/`, `services/`, `models/`, `repositories/`, `lib/`, `src/`, `app/`, `api/`, `pages/`, `components/`. You're building a map.

**Find entry points.** Look for the main server file, router definitions, or URL mapping. Per framework:
- Rails: `config/routes.rb`
- Express/Fastify: `app.ts`, `server.ts`, `src/routes/index.ts`
- Django: `urls.py`
- Next.js: `app/` or `pages/` directories
- Go: `main.go` and handler registration files

Understanding the routing structure tells you where new endpoints or pages would live.

**Read a similar existing feature.** This is often the most valuable step. Find a feature comparable to the one being planned and read its implementation end-to-end — what files it touches, how it moves through the layers. This shows the actual conventions the team follows, not just the abstract architecture. For example, if planning "notifications", find how the existing "email sending" feature is structured.

**Identify cross-cutting concerns.** Look for: auth middleware and permission checks, validation patterns, error handling conventions, logging/tracing, and API response shapes. New features almost always participate in these systems.

Calibrate the depth of exploration to the scope. A small focused app needs less exploration than a large layered codebase. The goal: enough understanding to identify every file that will need to change and every new file to create.

---

## Step 3: Identify the database schema

Schema understanding is often the most consequential part of planning. Getting this wrong leads to migrations that break existing queries or features that can't store what they need to.

Look for schema definitions in this order:

1. `prisma/schema.prisma` — Prisma
2. `db/schema.rb` — Rails ActiveRecord schema dump
3. `migrations/` or `db/migrate/` — SQL migration files (read the most recent ones to understand current state)
4. `*.sql` files at the project root or in a `sql/` or `database/` directory
5. ORM model files: `models/*.py` (Django/SQLAlchemy), `app/models/*.rb` (Rails), `src/models/*.ts` (TypeORM/Sequelize), `src/entities/*.ts` (TypeORM)
6. `drizzle/` or `drizzle.config.ts` — Drizzle ORM
7. `schema/` directories — GraphQL schemas, JSON schemas, or custom schema files

Read enough to understand: what entities exist, how they relate, what fields they have, and what indexes or constraints are in place. For migration-based schemas, the current state is the accumulation of all migrations — recent ones plus model files will give you current state.

Note the tables and columns directly relevant to the feature. Also note what does NOT exist yet — those gaps are where new migrations will be needed.

---

## Step 4: Ask clarifying questions (if needed)

After exploration you will likely have discovered ambiguities. Surface them concisely — no more than 5 questions in a single pass. Prioritize the ones that would most change the implementation.

Frame questions as: "While exploring the codebase I found X, which raises a question about your requirements: [question]." This shows you've done the work and grounds the question in something concrete.

If the requirements are complete enough to proceed without answers — feature is small and well-defined, or there's an obvious right approach — skip this step. Tell the user you're proceeding and that they can course-correct if anything looks wrong.

---

## Step 5: Produce the Refined Requirements document

Refined requirements translate the user's intent into engineering-level specificity, using what you learned from the codebase to fill in the gaps. Write this for humans — clear and direct.

Use this exact structure:

```
## Refined Requirements: [Feature Name]

### Summary
One or two sentences on what the feature does and why it matters.

### Functional Requirements
Numbered list. Each item must be specific enough to be testable.

Bad: "Users can log in with Google."
Good: "A user with no existing account who clicks 'Continue with Google' on /login
is redirected to Google OAuth, returned to /dashboard with a new session, and a
users row is created with provider='google' and the Google sub as external_id."

### Out of Scope
What this feature explicitly does NOT include in this iteration. Prevents scope
creep and signals intent clearly.

### Assumptions
Things assumed based on codebase exploration or common sense. The user should
validate these before implementation begins.

### Open Questions
Ambiguities requiring a decision before implementation can be completed.
Omit this section if there are none.
```

---

## Step 6: Produce the Implementation Plan

The implementation plan is the engineering artifact — specific enough that a developer could start writing code from it without additional research. Vague items like "update the user service" are not acceptable; each item should name the actual file, function, or table.

Use this exact structure:

```
## Implementation Plan: [Feature Name]

### Overview
2–3 sentences on the overall approach: what pattern this follows, what the key
architectural decisions are, and roughly how many files are involved.

### Database Changes
For each new or modified table:
- Table name and whether it is new or modified
- New columns: name, type, nullable, default, index if applicable
- New tables: full column list, primary key, foreign keys, indexes
- Migration approach: note whether the migration is additive (safe to deploy
  before code) or requires a coordinated deploy
If no DB changes are needed, say so explicitly.

### Backend Changes

#### New files
List each new file to create with its path and a sentence on what it does.

#### Modified files
For each existing file to modify:
- File path
- What changes: which functions are modified or added and what the change is

#### API changes
For each new or modified endpoint:
- Method + path
- Request shape (key fields)
- Response shape (key fields)
- Auth/permission requirements

### Frontend Changes
Omit this entire section if the codebase has no frontend or the feature is
backend/API-only.

#### New files
Same format as backend new files.

#### Modified files
Same format as backend modified files.

#### State / data fetching
What new queries, mutations, or store slices are needed.

### Tests
For each layer of the stack, list specific test cases to add:
- File the tests live in
- Scenarios to cover: happy path, edge cases, error cases

### Migration & Rollout Steps
Ordered sequence of steps to safely ship this feature. Call out any steps that
must be coordinated (e.g., migration before code deploy) or carry risk.

Example:
1. Write and review migration — confirm it is additive and safe to run first
2. Merge and deploy migration to staging; verify no errors
3. Deploy backend changes
4. Deploy frontend changes
5. Smoke test end-to-end on staging
6. Deploy to production

### Risks & Gotchas
Things that could go wrong or require special attention:
- Performance risks (queries that will do full table scans, N+1 patterns)
- Backward-compatibility concerns
- Shared abstractions touched by this change that other features depend on
- Anything requiring coordination across teams or services
```

---

## Tone and calibration

Adjust depth to the audience. A senior engineer may want dense, fast output. A less experienced developer may need more explanation. Always be specific — "add a migration creating the `user_oauth_providers` table with columns: `id UUID PRIMARY KEY`, `user_id UUID REFERENCES users(id)`, `provider VARCHAR(50)`, `external_id VARCHAR(255)`, `UNIQUE(provider, external_id)`" is always better than "add a migration."

If the codebase is large and you genuinely cannot explore all of it, be transparent — name what you looked at and what you could not see. A confident plan built on incomplete information is worse than a tentative plan that surfaces its own gaps.

---

## Output delivery

Deliver both documents in the same response, in this order:
1. Refined Requirements
2. `---` (horizontal rule)
3. Implementation Plan

After delivering, tell the user: "Review the requirements first — if anything looks wrong, correct it before acting on the implementation plan. The plan is derived from those requirements, so errors there propagate forward."
