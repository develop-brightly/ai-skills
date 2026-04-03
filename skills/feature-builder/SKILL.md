---
name: feature-builder
description: >
  Plan and scope new features using a collaborative 6-agent team: PM, Architect, Security, UX, Developer, and QA
  who communicate with each other and write structured markdown artifacts. Use when a user wants to
  implement something new, understand what needs to change to add a feature, write a technical spec,
  identify which files or database tables need to change, or translate business requirements into code.
  Trigger on: "I want to add X", "how would I implement Y", "what would it take to build Z",
  "help me plan this feature", "write the code for this feature", or any time a user describes a product
  idea and wants to understand or execute the engineering work. Do NOT use for quick how-to questions
  (e.g., "how do I use this API?", "what does this function do?", "show me an example of X").
  When in doubt, use this skill — the structured pipeline prevents costly rework from missed edge cases,
  schema gaps, or untested code.
---

# Feature Builder — Collaborative Agent Team

Turns vague feature requirements into refined specs, system designs, security reviews, UX specs, working code, and QA validation — using six specialized agents that can communicate with each other and write persistent markdown artifacts.

## What gets produced

All artifacts are written as markdown files to a `.feature-plan/` directory inside the working directory:

| File | Agent | Contents |
|---|---|---|
| `requirements.md` | PM | Refined requirements spec |
| `system-design.md` | Architect | System design document |
| `security-review.md` | Security | Threat model, OWASP review, implementation guidance |
| `ux-spec.md` | UX | User flows, component specs, copy, accessibility requirements |
| `implementation-summary.md` | Developer | Summary of files created/modified |
| `qa-report.md` | QA | Validation report and test cases |

---

## Agent Collaboration Model

This is not a strictly sequential hand-off pipeline. Agents are named and can send messages to each other during their work. This allows them to surface ambiguities and resolve design conflicts without stalling.

```
              ┌──────────────────────────────────────────────┐
              │              Feature Planner                 │
              │                                              │
  requirements│      ┌──────────┐                           │
  ────────────┼──────►  PM      │◄──────────────────────────┼── Architect asks PM to
              │      │  Agent   │   clarify req?             │   clarify ambiguous req
              │      └────┬─────┘                           │
              │           │ requirements.md                  │
              │      ┌────▼─────┐                           │
              │      │ Architect│◄──────────────────────────┼── Developer flags
              │      │  Agent   │   design gap?              │   design conflict
              │      └────┬─────┘                           │
              │           │ system-design.md                 │
              │   ┌───────┴────────┐  (parallel)            │
              │ ┌─▼──────┐  ┌─────▼──┐                     │
              │ │Security│  │  UX    │◄────────────────────┼── UX asks PM to
              │ │ Agent  │  │ Agent  │  clarify UX req?     │   clarify behavior
              │ └─┬──────┘  └────┬───┘                     │
              │   │security- │ux-spec.md                    │
              │   │review.md │                              │
              │   └───────┬──┘                              │
              │      ┌────▼─────┐                           │
              │      │Developer │◄──────────────────────────┼── QA flags
              │      │  Agent   │   code gap?                │   missing impl
              │      └────┬─────┘                           │
              │           │ implementation-summary.md        │
              │      ┌────▼─────┐                           │
              │      │  QA      │                           │
              │      │  Agent   │                           │
              │      └────┬─────┘                           │
              │           │ qa-report.md                     │
              └──────────────────────────────────────────────┘
```

Agents communicate via `SendMessage`. Each agent is assigned a name when spawned. The orchestrator waits for each stage to complete before spawning the next — but agents within a stage can exchange messages freely.

**Agent lifecycle**: All agents are spawned in **background mode** and remain reachable via `SendMessage` until Step 7 completes. An agent that has finished writing its primary artifact should enter a waiting state — do not terminate. It will receive follow-up messages from downstream agents and should respond before going idle again.

---

## Step 1: Intake and configure

Read any referenced files. Vague requirements are fine — the PM agent's job is to elaborate them.

**Ask the user two questions before proceeding:**

**Question 1 — Which agents should run?**

Use `ask_user` with the following choices (allow multi-select by asking once — the user can describe what they want):

```
Which specialist agents should be included in this pipeline?
choices: ["All agents (Security + UX) — Recommended", "Security Agent only", "UX/Design Agent only", "PM + Architect only (no specialist agents)"]
```

PM and Architect always run regardless of selection.

If the user doesn't answer or says "default" / "all", include Security and UX.

**Question 2 — Do you want to review before implementation?**

Use `ask_user` with the following choices:

```
After planning is complete, would you like to review the artifacts before the Developer writes any code?
choices: ["Yes, pause for review before implementation (Recommended)", "No, run the full pipeline automatically"]
```

If the user doesn't answer, default to pausing for review.

Store both answers — they shape the rest of the pipeline.

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

## Step 2a: Resolve PM open questions (if any)

After the PM agent completes, read `requirements.md`. Check whether it contains an **Open Questions** section with unresolved items.

If open questions exist, present them to the user using `ask_user` (freeform):

```
The PM identified the following open questions that could affect the design. Please answer them before the Architect begins:

[list the questions from requirements.md]
```

Once the user answers, spawn the PM agent briefly to update `requirements.md` with the answers and remove the Open Questions section (or mark them resolved). Wait for the update to complete.

If there are no open questions, proceed directly to Step 3.

---

## Step 3: Spawn the Architect Agent

Read `agents/architect.md`. Spawn a subagent named **`architect`** with:
- Path to `requirements.md`
- The working directory / repo path
- The output path: `.feature-plan/system-design.md`
- The name of the PM agent it can message for requirement clarification: **`pm`**

The Architect reads the requirements, explores the codebase deeply, and writes `system-design.md`. If it needs to clarify a requirement, it sends a message to the PM agent (which is still reachable) and waits for a reply. Wait for the Architect to complete before continuing.

---

## Step 4: Spawn specialist agents (conditional on Step 1 selections)

Only spawn the agents the user selected. Spawn any selected agents **at the same time** — they work independently and do not depend on each other.

**Security agent** *(if selected)* — Read `agents/security.md`. Spawn a subagent named **`security`** with:
- Path to `requirements.md`
- Path to `system-design.md`
- The working directory / repo path
- The output path: `.feature-plan/security-review.md`
- The name of the Architect agent it can message for design changes: **`architect`**

**UX agent** *(if selected)* — Read `agents/ux.md`. Spawn a subagent named **`ux`** with:
- Path to `requirements.md`
- Path to `system-design.md`
- The working directory / repo path
- The output path: `.feature-plan/ux-spec.md`
- The name of the PM agent it can message for UX clarifications: **`pm`**

If neither agent was selected, skip directly to the human review gate (Step 4a).

Wait for all selected agents to complete.

---

## Step 4a: Human review gate

Present the planning artifacts to the user for review. Read and display each completed artifact in order:

1. **Refined Requirements** (from `requirements.md`)
2. `---`
3. **System Design** (from `system-design.md`)
4. `---` *(if Security ran)*
5. **Security Review** (from `security-review.md`) — if recommendation is **BLOCKED**, call this out prominently before asking for approval
6. `---` *(if UX ran)*
7. **UX Spec** (from `ux-spec.md`)

Then use `ask_user` with the following choices:

```
Planning is complete. Would you like to proceed?
choices: ["Approve and proceed to implementation", "Request changes to the plan", "Stop here — planning artifacts only, no implementation needed"]
```

- If **"Approve and proceed to implementation"**: proceed to Step 5.
- If **"Request changes"**: ask the user what to revise using `ask_user` (freeform). Apply the requested changes by re-running the relevant agent(s) with updated instructions, then return to Step 4a.
- If **"Stop here"**: skip to Step 7 (final deliverables), omitting implementation and QA.

**Skip this gate** if the user chose option B ("run automatically") in Step 1. In that case, if the Security review is BLOCKED, still pause and surface the issue — never auto-proceed past a BLOCKED security recommendation.

---

## Step 5: Spawn the Developer Agent

Read `agents/developer.md`. Spawn a subagent named **`developer`** with:
- Path to `requirements.md`
- Path to `system-design.md`
- Path to `security-review.md` *(only if Security agent ran)*
- Path to `ux-spec.md` *(only if UX agent ran)*
- The working directory / repo path
- The output path: `.feature-plan/implementation-summary.md`
- The name of the Architect agent it can message for design questions: **`architect`**

The Developer implements the code and writes `implementation-summary.md`. If it discovers a conflict or gap in the system design, it sends a message to the Architect for clarification before proceeding. Wait for the Developer to complete.

---

## Step 6: Spawn the QA Agent

Read `agents/qa.md`. Spawn a subagent named **`qa`** with:
- Path to `requirements.md`
- Path to `system-design.md`
- Path to `security-review.md` *(only if Security agent ran)*
- Path to `ux-spec.md` *(only if UX agent ran)*
- Path to `implementation-summary.md`
- The working directory / repo path
- The output path: `.feature-plan/qa-report.md`
- The names of agents it can message for escalations: **`pm`**, **`architect`**, **`developer`**

The QA agent reads all implemented files, validates coverage, runs tests if possible, and writes `qa-report.md`. Wait for it to complete.

---

## Step 7: Present final deliverables

Read all completed artifact files and deliver them in one response. Include only the artifacts that were actually produced:

1. **Refined Requirements** (from `requirements.md`) — always present
2. `---`
3. **System Design** (from `system-design.md`) — always present
4. `---` *(if Security ran)*
5. **Security Review** (from `security-review.md`)
6. `---` *(if UX ran)*
7. **UX Spec** (from `ux-spec.md`)
8. `---` *(if implementation ran)*
9. **Implementation Summary** (from `implementation-summary.md`)
10. `---` *(if QA ran)*
11. **QA Validation Report** (from `qa-report.md`)

Then tell the user which artifacts were saved to `.feature-plan/` and note:
- If implementation ran: "The QA report flags any gaps or issues to address before shipping."
- If stopped at planning: "Implementation was not run. Approve the plan above and ask me to proceed when ready."

---

## Calibration notes

- **Depth**: Scale exploration depth to codebase size. A small focused repo needs less investigation than a layered monolith.
- **Sequential stages**: Agents run in order — each stage depends on the prior output files. Within each stage, agents use parallel tool calls freely to speed up exploration.
- **Failure handling**: After each agent completes, verify its output artifact exists and is non-empty before proceeding. If an artifact is missing or empty, do not continue — surface the failure to the user with the agent name and last known status, and ask whether to retry or abort. Never pass a missing path to a downstream agent.
- **Collaboration discipline**: Agents should reach out to each other only when genuinely blocked by an ambiguity that would materially change their output — not for minor questions they can resolve themselves with reasonable assumptions.
- **Scope discipline**: Developer and QA agents must not exceed the scope defined in `requirements.md`. If they identify necessary changes outside scope, they surface them as recommendations, not implementations.
- **Transparency**: If an agent cannot fully explore the codebase, it names what it looked at and what it could not see. A plan built on incomplete information should say so.
