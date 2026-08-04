# Development Workflow Diagram

This diagram is the authoritative reference for the RHDP Publishing House development phase.
Source of truth: `rhpds/rhdp-publishing-house` → `docs/superpowers/specs/2026-06-29-ph-skills-consolidation-architecture.md`

## Mermaid Diagram

```mermaid
flowchart TD
    User([User]) --> PH["/rhdp-publishing-house\nSKILL"]

    PH --> Intake["intake phase\nrhdp-publishing-house:intake\n02-discovery → 03-design-doc\n→ 04-module-outlines\n→ 05-infrastructure → 06-finalize"]

    Intake --> Dev["development phase\nrhdp-publishing-house:development"]

    Dev --> S1["Step 1: Scaffold check\nPREREQUISITE"]
    S1 --> CR["Run showroom:config-reviewer"]
    CR -->|PASS| S1b["Step 1b: Module status\nvalidation"]
    CR -->|FAIL| Issues["Report issues to user"]
    Issues -->|help me / fix it| CH["invoke showroom:config-helper\nAndrew, RHDPCD-172"]
    Issues -->|I'll handle it| STOP1["STOP\nUser scaffolds manually"]

    S1b --> InProgress{"Any in_progress?"}
    InProgress -->|yes| Warn["Warn user\nask to continue"]
    InProgress -->|no| AllComplete{"All complete\n+ 'write'?"}
    AllComplete -->|yes| Suggest["Suggest 'edit instead'"]
    AllComplete -->|no| S2["Step 2: Dispatch"]

    S2 -->|write| Writer["writer.md\nreads: spec.yaml + design.md\n+ module outline\npresents plan → waits for approval"]
    Writer --> MWH["[Task] showroom:module-writing-helper\nAGENT"]
    MWH --> Status["status: not_started\n→ in_progress → complete\nSequential: N blocked\nuntil 1..N-1 complete"]

    S2 -->|edit| Editor["editor.md"]
    Editor --> MR["[Task] showroom:module-reviewer\nAGENT"]
    MR --> Checks["SA-1→RS-2\nspec alignment checks"]

    S2 -->|automate| Auto["automation.md"]
    Auto -->|ansible| AH["[Skill] ansible-helper\nFUTURE — RHDPCD-110"]
    Auto -->|gitops| GH["[Skill] gitops-helper\nFUTURE — RHDPCD-111"]
```

## ASCII Diagram

```
User
  │
  └─ /rhdp-publishing-house (SKILL)
      │
      ├─ intake phase ─────────────────────────────────────────────┐
      │   rhdp-publishing-house:intake                             │
      │   └─ 02-discovery → 03-design-doc → 04-module-outlines    │
      │      → 05-infrastructure → 06-finalize                    │
      └────────────────────────────────────────────────────────────┘
      │
      ├─ development phase ────────────────────────────────────────┐
      │   rhdp-publishing-house:development                        │
      │                                                            │
      │   Step 1: Scaffold check (PREREQUISITE)                    │
      │   └─ Run showroom:config-reviewer                          │
      │      ├─ PASS → proceed to Step 1b                          │
      │      └─ FAIL → report issues to user                       │
      │               └─ User says "help me" / "fix it"            │
      │                  → invoke showroom:config-helper            │
      │                    (Andrew, RHDPCD-172)                     │
      │               └─ User says "I'll handle it"                │
      │                  → STOP. User scaffolds manually.           │
      │                                                            │
      │   Step 1b: Module status validation                        │
      │   └─ Read spec.yaml module statuses                        │
      │      ├─ Any in_progress? → warn user, ask to continue      │
      │      ├─ All complete + "write"? → "All done, edit instead?"│
      │      └─ Otherwise → proceed to Step 2                      │
      │                                                            │
      │   Step 2: Dispatch                                         │
      │   ├─ "write"  → writer.md                                  │
      │   │   ├─ reads: spec.yaml + design.md + module outline     │
      │   │   ├─ presents plan → waits for approval                │
      │   │   ├─ [Task] showroom:module-writing-helper (AGENT)     │
      │   │   └─ status: not_started → in_progress → complete      │
      │   │       └─ Sequential: N blocked until 1..N-1 complete   │
      │   │                                                        │
      │   ├─ "edit"   → editor.md                                  │
      │   │   ├─ [Task] showroom:module-reviewer (AGENT)           │
      │   │   └─ SA-1→RS-2 spec alignment checks                   │
      │   │                                                        │
      │   └─ "automate" → automation.md                            │
      │       ├─ [Skill] ansible-helper  (FUTURE — RHDPCD-110)     │
      │       └─ [Skill] gitops-helper   (FUTURE — RHDPCD-111)     │
      └────────────────────────────────────────────────────────────┘
```

## Legend

| Marker | Meaning |
|---|---|
| `[Task]` | Agent invocation via Task tool with `subagent_type` parameter |
| `[Skill]` | Skill invocation via Skill tool |
| `FUTURE` | Not yet built — dispatch path is wired but destination skill does not exist yet |

**Note:** FTL (`ftl:*`) is NOT in this diagram. Mitesh owns FTL validation separately as a standalone workflow.
