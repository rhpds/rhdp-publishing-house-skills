# Development Workflow Diagram

This diagram is the authoritative reference for the RHDP Publishing House development phase, after
the Phase 3 refactor that stripped the `development` skill down to scaffolding, module status
tracking, and Central submission. Writing, reviewing, and automation are optional standalone helper
skills the author invokes directly — `development` no longer dispatches to them.

## Mermaid Diagram

```mermaid
flowchart TD
    User([User]) --> PH["/rhdp-publishing-house\nSKILL"]

    PH --> Intake["intake phase\nrhdp-publishing-house:intake\n02-discovery to 03-design-doc\nto 04-module-outlines\nto 05-infrastructure to 06-finalize"]

    Intake --> Dev["development phase\nrhdp-publishing-house:development"]

    Dev --> S1["Step 1: Readiness check\nrecompute completeness live\nno saved 'complete' flag"]
    S1 -->|all pass, author confirms| Submit["run ph-development.py"]
    S1 -->|checks fail or author declines| S2["Step 2: Scaffold check gate"]

    S2 --> CR["Run procedures/config-reviewer.md"]
    CR -->|PASS| S2b["Step 2b: Module status\nvalidation gate"]
    CR -->|FAIL| Issues["Report issues to user"]
    Issues -->|help me / fix it| CH["procedures/config-helper.md"]
    Issues -->|I'll handle it| STOP1["STOP\nUser scaffolds manually"]

    S2b --> InProgress{"Any in_progress?"}
    InProgress -->|yes| Warn["Warn user\nask to continue"]
    InProgress -->|no| AllComplete{"All complete\n+ 'write'?"}
    AllComplete -->|yes| Suggest["Suggest 'edit instead'"]
    AllComplete -->|no| S3["Step 3: Module status\nmanagement"]

    S3 -->|"start module N"| InProg["status: not_started -> in_progress"]
    S3 -->|"module N is done"| Complete["verify .adoc exists\nstatus -> complete\nph-task-complete.py"]

    S3 --> S4["Step 4: Dispatch"]
    S4 -->|scaffold/config| CH2["procedures/config-helper.md\nprocedures/config-reviewer.md"]
    S4 -->|write/edit/automate phrasing| Redirect["Redirect message:\nuse an optional helper skill"]
    S4 -->|nothing specific| Dashboard["Development Dashboard\nstatus read from spec.yaml"]

    Redirect -.->|author invokes directly, not dispatched| WriterHelper["rhdp-publishing-house:writer-helper\nOPTIONAL, standalone"]
    Redirect -.->|author invokes directly, not dispatched| ReviewerHelper["rhdp-publishing-house:reviewer-helper\nOPTIONAL, standalone"]
    Redirect -.->|author invokes directly, not dispatched| AutomationHelper["rhdp-publishing-house:automation-helper\nOPTIONAL, standalone"]

    WriterHelper --> MWH["[Task] rhdp-publishing-house:module-writing-helper\nAGENT"]
    ReviewerHelper --> MR["[Task] rhdp-publishing-house:module-reviewer\nAGENT"]
    AutomationHelper -->|ansible| AH["[Skill] ansible-helper\nFUTURE - RHDPCD-110"]
    AutomationHelper -->|gitops| GH["[Skill] gitops-helper\nFUTURE - RHDPCD-111"]
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
      +- development phase (SCAFFOLDING ONLY) ---------------------------+
      |   rhdp-publishing-house:development                              |
      |                                                                  |
      |   Step 1: Readiness check (recomputed live, no saved flag)       |
      |   +- all pass + author confirms -> run ph-development.py         |
      |   +- otherwise -> Step 2                                        |
      |                                                                  |
      |   Step 2: Scaffold check gate                                    |
      |   +- Run procedures/config-reviewer.md                          |
      |      +- PASS -> proceed to Step 2b                              |
      |      +- FAIL -> report issues to user                           |
      |               +- "help me" -> procedures/config-helper.md       |
      |               +- "I'll handle it" -> STOP                       |
      |                                                                  |
      |   Step 2b: Module status validation gate                        |
      |                                                                  |
      |   Step 3: Module status management (AUTHORITATIVE, in SKILL.md) |
      |   +- "start module N" -> status: in_progress                    |
      |   +- "module N is done" -> verify .adoc exists, status: complete|
      |      -> ph-task-complete.py                                   |
      |                                                                  |
      |   Step 4: Dispatch                                              |
      |   +- scaffold/config -> config-helper.md / config-reviewer.md   |
      |   +- write/edit/automate phrasing -> redirect to optional helper|
      |   +- nothing specific -> Development Dashboard                  |
      +---------------------------------------------------------------+
      |
      +- optional helper skills (NOT dispatched by development) --------+
      |   rhdp-publishing-house:writer-helper                            |
      |   +- reads spec.yaml + design.md + module outline               |
      |   +- presents plan -> waits for approval                        |
      |   +- [Task] rhdp-publishing-house:module-writing-helper (AGENT) |
      |   +- may set status: in_progress (bookkeeping only)             |
      |   +- never sets status: complete, never submits to Central      |
      |                                                                  |
      |   rhdp-publishing-house:reviewer-helper                          |
      |   +- [Task] rhdp-publishing-house:module-reviewer (AGENT)       |
      |   +- SA-1 -> RS-2 spec alignment checks                         |
      |                                                                  |
      |   rhdp-publishing-house:automation-helper                       |
      |   +- ansible -> [Skill] ansible-helper (FUTURE - RHDPCD-110)    |
      |   +- gitops  -> [Skill] gitops-helper  (FUTURE - RHDPCD-111)    |
      +---------------------------------------------------------------+
```

## Legend

| Marker | Meaning |
|---|---|
| `[Task]` | Agent invocation via Task tool with `subagent_type` parameter |
| `[Skill]` | Skill invocation via Skill tool |
| `FUTURE` | Not yet built — dispatch path is wired but destination skill does not exist yet |
| `OPTIONAL, standalone` | Not dispatched by `development` — the author invokes it directly whenever they want |

**Note:** FTL (`ftl:*`) is NOT in this diagram. Mitesh owns FTL validation separately as a standalone workflow.
