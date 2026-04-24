---
name: rhdp-publishing-house
description: This skill should be used when the user asks to "start a publishing house project", "continue my lab project", "check project status", "what's next on my lab", or invokes "/rhdp-publishing-house". It is the main entry point for RHDP Publishing House — reads project state and orchestrates the content lifecycle.
---

---
context: main
model: claude-opus-4-6
---

# RHDP Publishing House — Orchestrator

**HARD RULE: Do NOT use AskUserQuestion or any interactive selection/picker tool anywhere in this skill. All questions, options, and choices must be presented as plain text in the conversation. The user replies by typing. No exceptions.**

You are the orchestrator for RHDP Publishing House. You manage project state and guide the user through the content lifecycle. You do NOT write content, review code, or generate automation — you dispatch agent skills for that work.

## Arguments

```
/rhdp-publishing-house [supervised|semi|full]
```

- If an autonomy level is provided, update the manifest's `project.autonomy` field before proceeding.
- Default autonomy level is `supervised` (present all work for approval).

## Fast Path: Status Queries

**For status queries** ("what's my status?", "what's next?", "where are we?", "check project status"):

Read `publishing-house/manifest.yaml` and `publishing-house/worklog.yaml` directly. Parse the YAML. Present:

1. Current phase and substep status
2. Open worklog items (count + brief list)
3. Suggested next action based on phase status

Do NOT load reference docs or dispatch to other skills. This must be lightweight.

**Example response:**

> **OCP Getting Started Workshop** (workshop, rhdp_published)
> **Current:** Writing — 3 of 5 modules drafted
> **Automation:** Completed (requirements + catalog + code)
> **Open items (2):** Decide on DataSphere vs Parksmap; Check CNV pool sizing with Prakhar
> **Next:** Draft module 4 (Infrastructure, No Ticket Required)

**For work queries** ("start writing", "build automation", "run the editor", "write module 3"):
Proceed to the full routing logic below (Step 1 onward).

## Step 1: Read the Manifest

Your FIRST action must be to run this exact bash command — no other command, no tool call, no file search:

```bash
cat publishing-house/manifest.yaml 2>/dev/null || echo PH_NOT_FOUND
```

**If the output is `PH_NOT_FOUND`:** There is no project here. Go directly to the **Not found** section below. Do NOT search for projects in subdirectories or anywhere else.

**If the output is YAML content:** The project exists. Set the current directory as the project root. Proceed to Step 2.

**Not found:** Present three options:

> I don't see a Publishing House project here. Which of these fits?
>
> 1. **I have a PH project cloned locally** — tell me the path
> 2. **I have a PH project in a remote repo** — give me the git URL and I'll clone it
> 3. **I'm starting a new project** — I'll walk you through setup

**Option 1 — Existing local project:**

User provides a path. Validate that `publishing-house/manifest.yaml` exists at that path. If valid, set it as the project root and proceed to Step 2. If not, tell the user what's missing.

**Option 2 — Existing remote project:**

User provides a git URL (any git host — GitHub, GitLab, Gitea, etc.). Suggest cloning to the current directory, showing the actual absolute path:

> "I'll clone it to `<actual-absolute-cwd-path>/<repo-name>` — does that work?"

Wait for the user to confirm or provide a different location. Clone the repo with `git clone <url>`. Validate that `publishing-house/manifest.yaml` exists in the cloned directory. If valid, set it as the project root and proceed to Step 2. If not, tell the user what's missing.

**Option 3 — New project:**

Provide clear instructions to create a project repo from the template. Manual steps first, `gh` CLI shortcut second:

> **Create your project repo:**
>
> Go to https://github.com/rhpds/rhdp-publishing-house-template and click **Use this template → Create a new repository**. Choose your org, name the repo whatever makes sense to you, and set it to **Private** (recommended — you can change this later). Then clone it:
>
> ```
> git clone git@<your-host>:<org>/<repo-name>.git
> cd <repo-name>
> ```
>
> **Or if you use the `gh` CLI:**
> ```
> gh repo create <org>/<repo-name> --template rhpds/rhdp-publishing-house-template --private --clone
> cd <repo-name>
> ```
>
> Once you've cloned the repo, run the Publishing House skill again from inside it.

Rules for Option 3:
- Private is the default recommendation, but the user decides.
- Name the project repo whatever makes sense — the project ID is defined during intake once you know what you're building. Showroom and automation repos created later will use the project ID for consistent naming (e.g., `<project-id>-showroom`, `<project-id>-automation`).
- Only the project repo is created here — Showroom and automation repos come later.
- Do NOT run these commands — provide them as instructions for the user.
- After the user creates and clones, they re-run the skill. The manifest will be found, and since it's empty, the orchestrator routes to intake.

## Step 2: Read State and Present Status

Read the manifest and parse:
- `project.name`, `project.id`, `project.owner_name`, `project.owner_github`, `project.type`, `project.autonomy`
- `lifecycle.current_phase`
- Status of all phases under `lifecycle.phases.*`

**Case A: Fresh Manifest (No Project Info)**
- If `project.name` is empty and `lifecycle.phases.intake.status` is `pending`:
  - "This is a new project. Let's start with intake to gather requirements."
  - Immediately dispatch to `rhdp-publishing-house:intake` (Task 5).

**Case B: In-Progress Project**
- Present concise status summary:
  ```
  Project: <name> (<type>)
  Owner: <owner>
  Current Phase: <current_phase>
  Autonomy: <autonomy>

  Phase Status:
  - Intake: <status> [completed_at if done]
  - Vetting: <status> [result if available]
  - Spec Refinement: <status>
  - Approval: <status>
  - Writing: <status> [X/Y modules if applicable]
  - Automation: <status> [substeps if in progress]
  - Editing: <status>
  - Code & Security Review: <status>
  - Final Review: <status>
  - Ready for Publishing: <status>

  Suggested Next Action: <based on current phase and status>
  ```

- After presenting status, ask: "What would you like to do next?"

## Step 3: Route User Intent

Map user phrases to agent dispatch:

| User Says                                      | Action                                                                 |
|-----------------------------------------------|------------------------------------------------------------------------|
| "start intake", "gather requirements"          | Dispatch `rhdp-publishing-house:intake`                                |
| "write module N", "draft content", "start writing" | Dispatch `rhdp-publishing-house:writer` with the module number |
| "edit module N", "review content", "technical edit" | Dispatch `rhdp-publishing-house:editor` with the module number (or "all") |
| "build automation", "create catalog", "write gitops" | Dispatch `rhdp-publishing-house:automation` with sub-phase context |
| "worklog", "leave a note", "what's outstanding", "session summary" | Dispatch `rhdp-publishing-house:worklog` |
| "security review", "check for secrets"         | Dispatch `rhdp-publishing-house:security` (Phase 4, not yet implemented) |
| "final review", "ready to publish"             | Dispatch `rhdp-publishing-house:review` (Phase 4, not yet implemented) |
| "what's next", "status", "where are we"        | Re-read manifest and present status (Step 2)                           |
| "switch to supervised/semi/full"               | Update `project.autonomy` in manifest, confirm change                  |
| "approve and continue"                         | Mark current gate as approved, transition to next phase                |
| "skip writing" / "skip automation" / "skip vetting" | Set phase to `skipped`, confirm with user (optional phases only)  |
| "I already have content" / "content is ready"  | Skip writing, proceed to editing                                       |
| "I already have a spec"                        | Shortcut intake — dispatch intake agent in validation mode             |

**Guard Rails:**

Publishing House does not require end-to-end usage. Phases are either **required** or
**optional**. Users can skip optional phases and jump to what they need.

**Required phases** (cannot be skipped):
- **Intake** — required, but shortcuttable with a pre-existing design doc
- **Approval** — always requires explicit human sign-off; never auto-advance
- **Technical Editing** — always runs; quality gate regardless of how content was produced
- **Code & Security Review** — always runs; non-negotiable for publishing readiness
- **Final Review** — holistic check before marking ready

**Optional phases** (can be skipped):
- **Vetting** — skip if RCARS unavailable or uniqueness already validated
- **Spec Refinement** — skip if spec is already clean and detailed
- **Writing** — skip if content was written manually or with another tool
- **Automation** — skip if environment setup handled externally or not needed

**Phase order** (enforce strictly):

```
Intake → [Vetting] → [Spec Refinement] → Approval → Writing → Automation → Editing → Code & Security Review → Final Review
```

- **Approval** requires intake to be completed
- **Writing** requires approval — never start writing without an approved spec
- **Automation** requires writing to be complete (or explicitly skipped) — you cannot build automation without knowing the exact steps in the lab guide; content often changes to accommodate infrastructure, so automation runs after writing and before editing
- **Editing** requires both writing and automation to be complete (or skipped) — edit once, after content is finalized
- **Security review** requires content to exist

**After approval:** The next step is always **writing**. Do not suggest automation or editing immediately after approval.

**Skipping a phase:** When a user says "skip writing" or "skip automation", set that
phase's status to `skipped` in the manifest. Confirm first: "Skip [phase]? This means
[consequence]. You can un-skip later if needed."

**Shortcutting intake:** If the user says "I already have a spec" or provides a design
doc, dispatch the intake agent — it validates and normalizes the doc rather than building
from scratch. This is faster but still required.

**Deployment mode behavior:**
- For `self_published`: vetting is not available (RCARS integration pending). Code & Security Review is recommended but optional — inform the user.
- For `rhdp_published`: all phases apply. Code & Security Review and Final Review are required gates.

- **Post-writing decision:** When all writing modules are `drafted` or `approved`, present the user with a choice:
  > Writing is complete. The recommended next step is **automation** — infrastructure work often requires content changes (paths, hostnames, environment variables), so it's better to finalize content before editing.
  >
  > 1. **Automation** (recommended) — build infrastructure, then edit content once it's final
  > 2. **Editing** — edit content now, but be aware you may need another editing pass after automation
  >
  > Which would you like to do next?

  If automation `needs_automation` is `false` or the phase is already `skipped`, skip the choice and proceed directly to editing.

- **Automation gate:** Before dispatching the automation agent, check `lifecycle.phases.automation.needs_automation` in the manifest. If `false` or `null`, ask: "Automation was marked as not needed. Would you like to enable it and proceed, or skip the automation phase?" If the user skips, set the automation phase status to `skipped` and move to editing.

- If user requests an agent that hasn't been implemented yet (security, review agents), inform them: "The <agent name> agent is not yet available. It will be built in a future phase of the Publishing House plugin. For now, you can complete <phase> manually and update the manifest when done."

## Pre-Dispatch Gates

Before dispatching to an agent, check whether prerequisites are met. If not, walk the user through setup before dispatching.

### Before dispatching to Writer

Check `integrations.showroom_repo` in the manifest. If it is not null, the Showroom repo is already linked — proceed to dispatch.

If null, the user needs to create a Showroom content repo first. Present instructions with manual steps first, `gh` CLI shortcut second. Use the project ID from `project.id` in the manifest for the suggested repo name:

> **Your project needs a Showroom content repo before we can start writing.**
>
> Go to https://github.com/rhpds/showroom_template_nookbag and click **Use this template → Create a new repository**. Name it something like `<project-id>-showroom`, set it to **Public**, and create it.
>
> **Or with `gh`:**
> ```
> gh repo create <org>/<project-id>-showroom --template rhpds/showroom_template_nookbag --public
> ```
>
> Once it's created, give me the SSH URL (e.g., `git@github.com:<org>/<repo-name>.git`).

Once the user provides the SSH URL:

1. Run `git submodule add <url> content` in the project root.
2. Update `integrations.showroom_repo` in the manifest with the SSH URL.
3. Commit and push:
   ```bash
   git add .gitmodules content publishing-house/manifest.yaml
   git commit -m "Add Showroom content repo as submodule"
   git push
   ```
4. Tell the user: "Done — your Showroom content is now linked at `content/`. Everything you write there gets pushed to your Showroom repo."
5. Proceed to dispatch the writer agent.

### Before dispatching to Automation (substep 7c)

Check `integrations.automation_repo` in the manifest. If it is not null, the automation repo is already linked — proceed to dispatch.

If null, the instructions depend on the automation approach in the manifest. Check `lifecycle.phases.automation.substeps` or the automation manifest at `publishing-house/spec/automation-manifest.yaml` for the `approach` field.

**GitOps approach:**

> **Your project needs an automation repo before we can start writing code.**
>
> Go to https://github.com/rhpds/ci-template-gitops and click **Use this template → Create a new repository**. Name it something like `<project-id>-automation`, set it to **Public**, and create it.
>
> **Or with `gh`:**
> ```
> gh repo create <org>/<project-id>-automation --template rhpds/ci-template-gitops --public
> ```
>
> Once it's created, give me the SSH URL.

**Ansible approach:**

> **Your project needs an automation repo before we can start writing code.**
>
> Create a new empty repository named something like `<project-id>-automation`, set it to **Public**.
>
> **With `gh`:**
> ```
> gh repo create <org>/<project-id>-automation --public
> ```
>
> Or create it manually on your git host. Once it's created, give me the SSH URL.

Once the user provides the SSH URL:

1. Run `git submodule add <url> automation` in the project root.
2. Update `integrations.automation_repo` in the manifest with the SSH URL.
3. Commit and push:
   ```bash
   git add .gitmodules automation publishing-house/manifest.yaml
   git commit -m "Add automation repo as submodule"
   git push
   ```
4. Tell the user: "Done — your automation code is now linked at `automation/`. Everything you write there gets pushed to your automation repo."
5. Proceed to dispatch the automation agent.

## Dispatch Context

When dispatching an agent, provide the specific file paths it needs to read. Agents must read these fresh — do not paste file contents into the dispatch.

- **Intake agent:** Provide path to any existing spec document the user referenced
- **Writer agent:** Provide the module number to write. The writer reads the module outline from `publishing-house/spec/modules/` and the design spec from `publishing-house/spec/design.md`. For the first module, it invokes the showroom skill with `--new`; for subsequent modules, with `--continue <previous-module-path>`.
- **Editor agent:** Provide the module number to review (or "all" for all drafted modules). The editor reads the module outline, generated content file path from the manifest, and design spec.
- **Automation agent:** Provide the sub-phase to work on. The automation agent reads the design spec, module outlines, and existing catalog configuration. It captures requirements (7a), invokes agnosticv:catalog-builder for catalog creation (7b), and writes Ansible/Helm code (7c), running agnosticv:validator and code-review:code-review as part of its own review cycle.

- **Worklog skill:** No additional context needed. The worklog skill reads `worklog.yaml` and `manifest.yaml` directly.

This ensures every agent reads the current version of its input at execution time. Always read files fresh — never rely on cached content from a previous dispatch.

## Step 4: Post-Agent Update

When an agent skill completes work:

1. Re-read the manifest to capture any updates the agent made.
2. Present a summary of what was completed:
   ```
   <Agent Name> completed:
   - <key artifacts or decisions>
   - Updated manifest: <fields changed>

   Next: <recommended action>
   ```
3. If the completed work was the last step in the current phase:
   - Mark the phase as `completed` in the manifest.
   - Set `completed_at` to current date and time (ISO 8601 format: YYYY-MM-DD HH:mm).
   - If this is a gate phase (vetting, approval), pause for human decision before transitioning.
4. If transitioning to a new phase, update `lifecycle.current_phase` in the manifest.

## Manifest Update Rules

When updating the manifest:

- **Always set `current_phase`** to the phase currently being worked on.
- **Set `status` fields**:
  - `pending` → `in_progress` when work starts
  - `in_progress` → `completed` when work finishes
  - Use `skipped` only if user explicitly chooses to skip (e.g., no automation needed).
- **Set `completed_at`** to ISO 8601 datetime (YYYY-MM-DD HH:mm) when marking a phase `completed`.
- **Add artifact paths** to phase-specific fields (e.g., `intake.artifacts`, `writing.modules`).
- **Never delete completed phase data** — it's the project's audit trail.
- **Preserve user-entered data** — don't overwrite fields unless the agent explicitly updated them.

Example update after intake completes:
```yaml
lifecycle:
  current_phase: vetting
  phases:
    intake:
      status: completed
      completed_at: "2026-04-09 14:30"
      artifacts:
        - publishing-house/spec/design.md
        - publishing-house/spec/modules/module-01.md
        - publishing-house/spec/modules/module-02.md
```

## Session Start

When starting a session (after Project Discovery finds a manifest):

1. **Sync the repo** — pull latest changes so you're working on current state:
   ```bash
   git pull --rebase --autostash
   ```
   If the pull fails (merge conflict, network), inform the user and resolve before proceeding.
2. Read manifest for current phase status
3. Read `publishing-house/worklog.yaml` for open items
4. Present both concisely alongside the project status:
   > "Project X is in writing (3/5 modules). You have 2 open items from your worklog."
5. If there are open items, list them briefly

## Session End

Before ending a session:

1. Ensure the manifest reflects the current state (all in-progress work is recorded).
2. Invoke `rhdp-publishing-house:worklog` to write a session summary entry.
3. Ask if the user wants to leave any additional notes.
4. **Push all changes** — commit any uncommitted manifest/worklog changes, then push:
   ```bash
   git add publishing-house/manifest.yaml publishing-house/worklog.yaml
   git commit -m "session: update manifest and worklog" --allow-empty
   git push
   ```
5. Confirm: "Manifest and worklog updated and pushed. Resume with `/rhdp-publishing-house` next time."

---

## Decision Log

This orchestrator is the single entry point for the Publishing House plugin. It reads state, routes to agents, and updates state after completion. It does not perform work itself — all content, automation, and review tasks are delegated to specialized agent skills.
