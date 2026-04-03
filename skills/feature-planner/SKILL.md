---
name: feature-planner
description: >
  Plan and scope new features using a 4-agent team: PM → Architect → Developer → QA.
  Use this skill whenever a user wants to implement something new, understand what needs
  to change to add a feature, write a technical spec, figure out which files or database
  tables need to change, or translate business requirements into working code. Trigger on
  phrases like: "I want to add X", "how would I implement Y", "what would it take to build Z",
  "help me plan this feature", "I need to scope out this change", "write the code for this
  feature", or any time a user describes a product idea and wants to understand or execute
  the engineering work involved. When in doubt, use this skill — the structured multi-agent
  pipeline prevents costly rework from missed edge cases, schema gaps, or untested code.
---

# Feature Planner — Agent Team

Turns vague feature requirements into refined specs, system designs, working code, and QA validation — by running four specialized agents in sequence, each handing its output to the next.

## Overview

The pipeline runs in four stages:

```
User requirements
      │
      ▼
  PM Agent          → Refined Requirements spec
      │
      ▼
  Architect Agent   → System Design doc (reads PM spec)
      │
      ▼
  Developer Agent   → Working code (reads PM spec + Architect design)
      │
      ▼
  QA Agent          → Test cases + validation report (reads all prior outputs)
```

Each agent is defined in `agents/`. Read the relevant file before spawning that agent.

---

## Step 1: Intake requirements

Determine how the user has provided requirements. They may have described the feature conversationally, pasted a ticket or spec inline, or referenced a file. If they referenced a file, read it now.

Vague requirements are fine — the PM agent's job is to elaborate them. Do not demand a detailed spec before starting.

---

## Step 2: Run the PM Agent

Read `agents/pm.md`, then spawn a subagent with those instructions plus:
- The raw requirements from the user
- The working directory / repo path

The PM agent will explore the codebase and produce a **Refined Requirements** document. Save its output before proceeding.

---

## Step 3: Run the Architect Agent

Read `agents/architect.md`, then spawn a subagent with those instructions plus:
- The Refined Requirements from the PM agent
- The working directory / repo path

The Architect agent will explore the codebase more deeply and produce a **System Design** document covering DB changes, new/modified files, API contracts, and architectural decisions. Save its output before proceeding.

---

## Step 4: Run the Developer Agent

Read `agents/developer.md`, then spawn a subagent with those instructions plus:
- The Refined Requirements from the PM agent
- The System Design from the Architect agent
- The working directory / repo path

The Developer agent writes the actual implementation: migrations, new source files, modified files. It must follow the design exactly and stay within the scope defined by the PM spec. Save its output (file paths and content written) before proceeding.

---

## Step 5: Run the QA Agent

Read `agents/qa.md`, then spawn a subagent with those instructions plus:
- The Refined Requirements from the PM agent
- The System Design from the Architect agent
- A list of files written/modified by the Developer agent
- The working directory / repo path

The QA agent reads the implemented code, runs or generates tests, checks each functional requirement for coverage, and produces a **Validation Report**. Save its output.

---

## Step 6: Present the final deliverables

Deliver all four artifacts in one response, in this order:

1. **Refined Requirements** (from PM)
2. `---`
3. **System Design** (from Architect)
4. `---`
5. **Implementation Summary** — which files were created/modified and what changed (from Developer)
6. `---`
7. **QA Validation Report** (from QA)

Then tell the user:

> "The PM spec drives everything downstream — if any requirement looks wrong, correct it before acting on the implementation. The QA report flags any gaps or issues to address before shipping."

---

## Calibration notes

- **Depth**: Scale exploration depth to codebase size. A small focused repo needs less investigation than a layered monolith.
- **Parallelism**: The four agents are sequential by design (each depends on the prior stage's output). Within each agent, use parallel subagents or tool calls freely to speed up exploration.
- **Scope discipline**: Developer and QA agents must not exceed the scope defined in the PM spec. If they identify necessary changes not in the spec, surface them as recommendations rather than acting on them.
- **Transparency**: If an agent cannot fully explore the codebase, it should name what it looked at and what it could not see. A confident plan built on incomplete information is worse than a transparent one that surfaces its own gaps.
