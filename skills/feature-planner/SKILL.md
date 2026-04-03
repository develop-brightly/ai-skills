---
name: feature-planner
description: >
  Plan and scope new features using a collaborative 4-agent team: PM, Architect, Developer, and QA
  who can communicate with each other and write structured markdown artifacts. Use this skill whenever
  a user wants to implement something new, understand what needs to change to add a feature, write a
  technical spec, figure out which files or database tables need to change, or translate business
  requirements into working code. Trigger on phrases like: "I want to add X", "how would I implement Y",
  "what would it take to build Z", "help me plan this feature", "I need to scope out this change",
  "write the code for this feature", or any time a user describes a product idea and wants to understand
  or execute the engineering work involved. When in doubt, use this skill — the structured collaborative
  agent pipeline prevents costly rework from missed edge cases, schema gaps, or untested code.
---

# Feature Planner — Collaborative Agent Team

Turns vague feature requirements into refined specs, system designs, working code, and QA validation — using four specialized agents that can communicate with each other and write persistent markdown artifacts.

## What gets produced

All artifacts are written as markdown files to a `.feature-plan/` directory inside the working directory:

| File | Agent | Contents |
|---|---|---|
| `requirements.md` | PM | Refined requirements spec |
| `system-design.md` | Architect | System design document |
| `implementation-summary.md` | Developer | Summary of files created/modified |
| `qa-report.md` | QA | Validation report and test cases |

---

## Agent Collaboration Model

This is not a strictly sequential hand-off pipeline. Agents are named and can send messages to each other during their work. This allows them to surface ambiguities and resolve design conflicts without stalling.

```
              ┌─────────────────────────────────────┐
              │         Feature Planner              │
              │                                      │
  requirements│      ┌──────────┐                   │
  ────────────┼──────►  PM      │◄──────────────────┼── Architect asks PM to
              │      │  Agent   │   clarify req?     │   clarify ambiguous req
              │      └────┬─────┘                   │
              │           │ requirements.md          │
              │      ┌────▼─────┐                   │
              │      │ Architect│◄──────────────────┼── Developer flags
              │      │  Agent   │   design gap?      │   design conflict
              │      └────┬─────┘                   │
              │           │ system-design.md         │
              │      ┌────▼─────┐                   │
              │      │Developer │◄──────────────────┼── QA flags
              │      │  Agent   │   code gap?        │   missing impl
              │      └────┬─────┘                   │
              │           │ implementation-summary.md│
              │      ┌────▼─────┐                   │
              │      │  QA      │                   │
              │      │  Agent   │                   │
              │      └────┬─────┘                   │
              │           │ qa-report.md             │
              └─────────────────────────────────────┘
```

Agents communicate via `SendMessage`. Each agent is assigned a name when spawned. The orchestrator waits for each stage to complete before spawning the next — but agents within a stage can exchange messages freely.

---

## Step 1: Intake requirements

Read any referenced files. Vague requirements are fine — the PM agent's job is to elaborate them.

Then create the output directory:
```bash
mkdir -p .feature-plan
```

---

## Step 2: Spawn the PM Agent

Read `agents/pm.md`. Spawn a subagent named **`pm`** with:
- The raw requirements from the user
- The working directory / repo path
- The output path: `.feature-plan/requirements.md`
- The name of the Architect agent it can message if blocked: **`architect`** (not yet running — tell PM to surface open questions in the doc instead)

The PM agent will explore the codebase and write `requirements.md`. Wait for it to complete.

---

## Step 3: Spawn the Architect Agent

Read `agents/architect.md`. Spawn a subagent named **`architect`** with:
- Path to `requirements.md`
- The working directory / repo path
- The output path: `.feature-plan/system-design.md`
- The name of the PM agent it can message for requirement clarification: **`pm`**

The Architect reads the requirements, explores the codebase deeply, and writes `system-design.md`. If it needs to clarify a requirement, it sends a message to the PM agent (which is still reachable) and waits for a reply. Wait for the Architect to complete before continuing.

---

## Step 4: Spawn the Developer Agent

Read `agents/developer.md`. Spawn a subagent named **`developer`** with:
- Path to `requirements.md`
- Path to `system-design.md`
- The working directory / repo path
- The output path: `.feature-plan/implementation-summary.md`
- The name of the Architect agent it can message for design questions: **`architect`**

The Developer implements the code and writes `implementation-summary.md`. If it discovers a conflict or gap in the system design, it sends a message to the Architect for clarification before proceeding. Wait for the Developer to complete.

---

## Step 5: Spawn the QA Agent

Read `agents/qa.md`. Spawn a subagent named **`qa`** with:
- Path to `requirements.md`
- Path to `system-design.md`
- Path to `implementation-summary.md`
- The working directory / repo path
- The output path: `.feature-plan/qa-report.md`
- The names of agents it can message for escalations: **`pm`**, **`architect`**, **`developer`**

The QA agent reads all implemented files, validates coverage, runs tests if possible, and writes `qa-report.md`. Wait for it to complete.

---

## Step 6: Present final deliverables

Read all four markdown files and deliver them in one response, in this order:

1. **Refined Requirements** (from `requirements.md`)
2. `---`
3. **System Design** (from `system-design.md`)
4. `---`
5. **Implementation Summary** (from `implementation-summary.md`)
6. `---`
7. **QA Validation Report** (from `qa-report.md`)

Then tell the user:

> "All four artifacts have been saved to `.feature-plan/`. The PM spec drives everything downstream — if any requirement looks wrong, correct it before acting on the implementation. The QA report flags any gaps or issues to address before shipping."

---

## Calibration notes

- **Depth**: Scale exploration depth to codebase size. A small focused repo needs less investigation than a layered monolith.
- **Sequential stages**: Agents run in order — each stage depends on the prior output files. Within each stage, agents use parallel tool calls freely to speed up exploration.
- **Collaboration discipline**: Agents should reach out to each other only when genuinely blocked by an ambiguity that would materially change their output — not for minor questions they can resolve themselves with reasonable assumptions.
- **Scope discipline**: Developer and QA agents must not exceed the scope defined in `requirements.md`. If they identify necessary changes outside scope, they surface them as recommendations, not implementations.
- **Transparency**: If an agent cannot fully explore the codebase, it names what it looked at and what it could not see. A plan built on incomplete information should say so.
