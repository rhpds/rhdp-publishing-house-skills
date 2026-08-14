# Development Workflow Diagram

This diagram is the authoritative reference for the RHDP Publishing House development phase.
The development skill provides a numbered dashboard for 4 workstreams (modules, automation,
e2e, health check) plus showroom config. Writing, reviewing, and automation are optional
standalone helper skills the author invokes directly.

## Mermaid Diagram

```mermaid
flowchart TD
    User([User]) --> PH["/rhdp-publishing-house\nSKILL"]

    PH --> Intake["intake phase\nrhdp-publishing-house:intake\n02-discovery to 03-design-doc\nto 04-module-outlines\nto 05-infrastructure to 06-finalize"]

    Intake --> Dev["development phase\nrhdp-publishing-house:development"]

    Dev --> S1["Step 1: Scaffold check gate"]
    S1 --> CR["Run procedures/config-reviewer.md"]
    CR -->|PASS| Dashboard
    CR -->|FAIL| Issues["Report issues to user"]
    Issues -->|option 1: help me| CH["procedures/config-helper.md"]
    Issues -->|option 2: I'll handle it| STOP1["STOP"]

    Dashboard["Step 2: Development Dashboard\n1. Modules (N/M complete)\n2. Automation\n3. E2E Tests\n4. Health Check\n5. Showroom Config"]

    Dashboard -->|all complete| Submit["Step 3: Submit\nph-development.py"]

    Dashboard -->|option 1| Modules["Module sub-menu\nstart/complete modules\nph-task-complete.py"]
    Dashboard -->|option 2| Auto["Automation sub-menu\nscaffold or populate"]
    Dashboard -->|option 3| E2E["E2E sub-menu\n1. Mark complete\n2. Back"]
    Dashboard -->|option 4| HC["Health Check sub-menu\n1. Mark complete\n2. Back"]
    Dashboard -->|option 5| Config["Showroom Config\n1. config-helper\n2. config-reviewer\n3. Back"]

    Auto -->|no automation/ dir| CH2["procedures/config-helper.md\nAutomation Scaffolding"]
    Auto -->|automation/ exists| AutoMenu["1. GitOps helper\n2. Ansible (placeholder)\n3. Mark complete"]
    AutoMenu -->|option 1| GH["[Skill] gitops-helper"]
    AutoMenu -->|option 2| AH["Ansible helper\nFUTURE - RHDPCD-110"]

    Modules -.->|author invokes directly| WriterHelper["rhdp-publishing-house:writer-helper\nOPTIONAL"]
    Modules -.->|author invokes directly| ReviewerHelper["rhdp-publishing-house:reviewer-helper\nOPTIONAL"]

    WriterHelper --> MWH["[Task] rhdp-publishing-house:module-writing-helper\nAGENT"]
    ReviewerHelper --> MR["[Task] rhdp-publishing-house:module-reviewer\nAGENT"]
```

## ASCII Diagram

```
User
  |
  +- /rhdp-publishing-house (SKILL)
      |
      +- intake phase --------------------------------------------------+
      |   rhdp-publishing-house:intake                                  |
      |   +- 02-discovery -> 03-design-doc -> 04-module-outlines        |
      |      -> 05-infrastructure -> 06-finalize                        |
      +-------------------------------------------------------------------+
      |
      +- development phase ----------------------------------------------+
      |   rhdp-publishing-house:development                              |
      |                                                                  |
      |   Step 1: Scaffold check gate                                    |
      |   +- Run procedures/config-reviewer.md                          |
      |      +- PASS -> proceed to Dashboard                            |
      |      +- FAIL -> report issues, offer numbered options            |
      |                                                                  |
      |   Step 2: Development Dashboard (numbered options)               |
      |   +- 1. Modules    -> module sub-menu (start/complete)           |
      |   +- 2. Automation -> config-helper scaffold / gitops-helper        |
      |   +- 3. E2E Tests  -> mark complete (placeholder)               |
      |   +- 4. Health Check -> mark complete (placeholder)              |
      |   +- 5. Showroom Config -> config-helper / config-reviewer       |
      |                                                                  |
      |   Step 3: Submission gate                                        |
      |   +- ALL 4 workstreams complete -> ph-development.py             |
      +---------------------------------------------------------------+
      |
      +- optional helper skills (NOT dispatched by development) --------+
      |   rhdp-publishing-house:writer-helper                            |
      |   +- [Task] rhdp-publishing-house:module-writing-helper (AGENT) |
      |                                                                  |
      |   rhdp-publishing-house:reviewer-helper                          |
      |   +- [Task] rhdp-publishing-house:module-reviewer (AGENT)       |
      |                                                                  |
      |   rhdp-publishing-house:automation-helper                       |
      |   +- gitops  -> [Skill] gitops-helper                            |
      |   +- ansible -> [Skill] ansible-helper (FUTURE - RHDPCD-110)    |
      +---------------------------------------------------------------+
```

## Legend

| Marker | Meaning |
|---|---|
| `[Task]` | Agent invocation via Task tool with `subagent_type` parameter |
| `[Skill]` | Skill invocation via Skill tool |
| `FUTURE` | Not yet built — dispatch path is wired but destination skill does not exist yet |
| `OPTIONAL` | Not dispatched by `development` — the author invokes it directly whenever they want |

**Note:** FTL (`ftl:*`) is NOT in this diagram. Mitesh owns FTL validation separately as a standalone workflow.
