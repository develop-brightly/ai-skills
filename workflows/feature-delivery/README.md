# Feature Delivery — End-to-End Agent Pipeline

Turns feature requirements into shipped, validated code — using six specialized agents that plan, design, review, implement, and QA in sequence, communicating with each other to resolve conflicts without stalling.

Unlike a planning-only workflow, `feature-delivery` goes all the way: every run produces working code and a QA validation with a ship/no-ship recommendation.

## What gets produced

All artifacts are written as markdown files to a `.feature-delivery/` directory inside the working directory:

| File | Agent | Contents |
|---|---|---|
| `requirements.md` | PM | Refined requirements with acceptance criteria and definition of done |
| `system-design.md` | Architect | System design with deployment ordering and rollback strategy |
| `security-review.md` | Security | Threat model, OWASP review, implementation guidance |
| `ux-spec.md` | UX | User flows, component specs, copy, accessibility requirements |
| `implementation-summary.md` | Developer | Summary of files created/modified |
| `qa-report.md` | QA | Validation report, test cases, and ship/no-ship recommendation |

---

## Agent Collaboration Model

Agents are named and can send messages to each other during their work. This allows them to surface ambiguities and resolve design conflicts without stalling.

```
              ┌──────────────────────────────────────────────┐
              │             Feature Delivery                 │
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
              │           │ qa-report.md (ship/no-ship)      │
              └──────────────────────────────────────────────┘
```

Agents communicate via `SendMessage`. Each agent is assigned a name when spawned. The orchestrator waits for each stage to complete before spawning the next — but agents within a stage can exchange messages freely.

**Agent lifecycle**: All agents are spawned in **background mode** and remain reachable via `SendMessage` until Step 6 completes. An agent that has finished writing its primary artifact should enter a waiting state — do not terminate. It will receive follow-up messages from downstream agents and should respond before going idle again.

---

## Step 1: Intake

Read any referenced files. Vague requirements are fine — the PM agent's job is to elaborate them.

**Ask the user one question before proceeding:**

```
After planning is complete, would you like to review the artifacts before the Developer writes any code?
choices: ["Yes, pause for review before implementation (Recommended)", "No, run the full pipeline automatically"]
```

If the user doesn't answer, default to pausing for review.

Store the answer — it shapes the pipeline at Step 4a.

Then create the output directory:
```bash
mkdir -p .feature-delivery
```

All six agents — PM, Architect, Security, UX, Developer, QA — always run in `feature-delivery`. There is no agent-selection step.

---

## Step 2: Spawn the PM Agent

Read `agents/pm.md`. Spawn a subagent named **`pm`** with:
- The raw requirements from the user
- The working directory / repo path
- The output path: `.feature-delivery/requirements.md`
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
- The output path: `.feature-delivery/system-design.md`
- The name of the PM agent it can message for requirement clarification: **`pm`**

The Architect reads the requirements, explores the codebase deeply, and writes `system-design.md`. If it needs to clarify a requirement, it sends a message to the PM agent (which is still reachable) and waits for a reply. Wait for the Architect to complete before continuing.

---

## Step 4: Spawn Security and UX agents (parallel)

Spawn both agents **at the same time** — they work independently and do not depend on each other.

**Security agent** — Read `agents/security.md`. Spawn a subagent named **`security`** with:
- Path to `requirements.md`
- Path to `system-design.md`
- The working directory / repo path
- The output path: `.feature-delivery/security-review.md`
- The name of the Architect agent it can message for design changes: **`architect`**

**UX agent** — Read `agents/ux.md`. Spawn a subagent named **`ux`** with:
- Path to `requirements.md`
- Path to `system-design.md`
- The working directory / repo path
- The output path: `.feature-delivery/ux-spec.md`
- The name of the PM agent it can message for UX clarifications: **`pm`**

Wait for both agents to complete.

---

## Step 4a: Human review gate

Present a summary of the planning artifacts to the user. Read each completed artifact in order and summarize the key points (do not dump the full content):

1. **Refined Requirements** — key acceptance criteria and definition of done
2. **System Design** — files to create/modify, DB changes, deployment steps
3. **Security Review** — overall risk level and recommendation; if BLOCKED, call this out prominently
4. **UX Spec** — user flows and key interaction decisions

Then use `ask_user` with the following choices:

```
Planning is complete. Ready to proceed to implementation?
choices: ["Yes — implement and deliver", "Request changes to the plan", "Stop here — planning artifacts only"]
```

- If **"Yes — implement and deliver"**: proceed to Step 5.
- If **"Request changes"**: ask the user what to revise using `ask_user` (freeform). Apply the requested changes by re-running the relevant agent(s) with updated instructions, then return to Step 4a.
- If **"Stop here"**: skip to Step 7 (final deliverables), omitting implementation and QA.

**Skip this gate** if the user chose to run automatically in Step 1. However, if the Security review is **BLOCKED**, always pause — never auto-proceed past a BLOCKED security recommendation.

---

## Step 5: Spawn the Developer Agent

Read `agents/developer.md`. Spawn a subagent named **`developer`** with:
- Path to `requirements.md`
- Path to `system-design.md`
- Path to `security-review.md`
- Path to `ux-spec.md`
- The working directory / repo path
- The output path: `.feature-delivery/implementation-summary.md`
- The name of the Architect agent it can message for design questions: **`architect`**

The Developer implements the code and writes `implementation-summary.md`. If it discovers a conflict or gap in the system design, it sends a message to the Architect for clarification before proceeding. Wait for the Developer to complete.

---

## Step 6: Spawn the QA Agent

Read `agents/qa.md`. Spawn a subagent named **`qa`** with:
- Path to `requirements.md`
- Path to `system-design.md`
- Path to `security-review.md`
- Path to `ux-spec.md`
- Path to `implementation-summary.md`
- The working directory / repo path
- The output path: `.feature-delivery/qa-report.md`
- The names of agents it can message for escalations: **`pm`**, **`architect`**, **`developer`**

The QA agent reads all implemented files, validates coverage, runs tests if possible, and writes `qa-report.md` — including a final ship/no-ship recommendation. Wait for it to complete.

---

## Step 7: Present final deliverables

Read the QA report's ship/no-ship recommendation and surface it prominently first:

> **Ship recommendation:** [READY TO SHIP / NEEDS FIXES / NEEDS REVIEW]

Then list all artifact files produced in `.feature-delivery/`:

1. **Refined Requirements** (`requirements.md`)
2. **System Design** (`system-design.md`)
3. **Security Review** (`security-review.md`)
4. **UX Spec** (`ux-spec.md`)
5. **Implementation Summary** (`implementation-summary.md`) — *(if implementation ran)*
6. **QA Validation Report** (`qa-report.md`) — *(if QA ran)*

If the recommendation is **NEEDS FIXES**: list the blocking issues from the QA report and tell the user what to resolve before shipping.

If the recommendation is **READY TO SHIP**: note any non-blocking items from the QA report the user may want to follow up on.

If stopped at planning (Step 4a "Stop here"): note that implementation was not run and the user can proceed by asking for implementation when ready.

---

## Calibration notes

- **This workflow ships code**: Unlike a planning-only workflow, `feature-delivery` is expected to produce working, production-ready implementation in every run. The QA agent's ship/no-ship verdict is the exit signal.
- **Sequential stages**: Agents run in order — each stage depends on the prior output files. Within each stage, agents use parallel tool calls freely to speed up exploration.
- **Failure handling**: After each agent completes, verify its output artifact exists and is non-empty before proceeding. If an artifact is missing or empty, surface the failure to the user with the agent name and last known status, and ask whether to retry or abort. Never pass a missing path to a downstream agent.
- **Collaboration discipline**: Agents should reach out to each other only when genuinely blocked — not for minor questions they can resolve with reasonable assumptions.
- **Scope discipline**: Developer and QA agents must not exceed the scope defined in `requirements.md`. If they identify necessary changes outside scope, they surface them as recommendations, not implementations.
- **Transparency**: If an agent cannot fully explore the codebase, it names what it looked at and what it could not see. A plan built on incomplete information should say so.
