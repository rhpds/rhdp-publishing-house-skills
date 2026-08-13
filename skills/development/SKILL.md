---
name: rhdp-publishing-house:development
description: This skill should be used when the user asks to "set up showroom", "configure showroom tabs", "create site.yml", "scaffold the showroom structure", "add a tab", "review my showroom config", "check site.yml", "validate ui-config.yml", "module N is done", "mark it complete", "what's next to develop", or "submit to central". Handles showroom scaffolding, config review, module status tracking, and submission to Central during the development stage.
context: main
---

# Development Agent

**RULE: If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately.** Show the error output to the author and say there was an issue calling the backend. Do not continue the skill.

You handle the development phase of the Publishing House lifecycle. After scaffolding, this skill
does exactly three things: scaffold the showroom structure, track module status in `spec.yaml`, and
submit to Central when the author says all content is complete. Writing, reviewing, and automation
are optional helper skills the author may use — or not — entirely at their own discretion.

## Tool Boundaries

**Do NOT use** Central API tools directly. You work locally: read files, write content, update spec.yaml.

**Do NOT use** MCP tools. All external interactions go through `publishing-house/tools/` scripts.

## Steps 1–3 — Pre-flight

Follow @rhdp-publishing-house/skills/common/pre-flight.md (Steps 1–3: verify project, read identity, check auth).

### Step 4 — Workflow check

**RULE: This sequence runs every invocation. No exceptions. No skipping. No reusing previous results.**

**4a.** Get workflow data:
```bash
python publishing-house/tools/ph-workflow-data.py
```
If this fails → set `offline_mode = true`, skip to Step 5.
If this succeeds → extract `workflow_id`. Set `offline_mode = false`.

**4b.** Get workflow state (skip if offline):
```bash
python publishing-house/tools/ph-workflow-state.py WORKFLOW_ID
```
If stage is not `development` → STOP. This is the only condition that stops the skill.
If offline → assume `development`.

**4c.** Sync (skip if offline):
```bash
python publishing-house/tools/ph-sync.py
```
Extract `unresolved_rejections` from the output. Commit any changes:
```bash
git add publishing-house/spec.yaml catalog-info.yaml
git diff --cached --quiet || git commit -m "feat: sync workflow data from Central API" 2>/dev/null || true
```

**4d.** If `unresolved_rejections` > 0 → show the unresolved rejection reasons to the author (read from `approval_checklist.content.rejections` and `approval_checklist.infra.rejections` in spec.yaml). Help the author address each one, then continue with normal development.

### Step 5 — Read project context

1. Read `publishing-house/spec.yaml` for project metadata and spec data
2. Read `publishing-house/spec/design.md` for the design spec
3. Read module outlines in `publishing-house/spec/modules/`

## Dispatch

### Step 1 — Scaffold check gate

Follow `procedures/config-reviewer.md` automatically against the project's content directory.

- **PASS** → proceed to Step 2
- **FAIL** → report the specific issues to the author, then ask:
  > "The showroom scaffold has issues that need to be resolved first. Would you like me to help fix them, or will you handle it?"
  - "help me" → follow `procedures/config-helper.md` to fix the scaffold, then proceed to Step 2
  - "I'll handle it" → STOP. Do not proceed until the author says the scaffold is ready.

### Step 2 — Development Dashboard

Read all workstream statuses from `spec.yaml` and present the dashboard. This runs every time —
it is the central hub of the development phase.

Read from `spec.yaml`:
- Module statuses from `spec.modules[*].status`
- `development.automation.status`
- `development.e2e.status`
- `development.healthCheck.status`

Present:

> **Development Dashboard**
>
> | # | Workstream | Status |
> |---|------------|--------|
> | 1 | Modules | N of M complete |
> | 2 | Automation | not_started / in_progress / complete |
> | 3 | E2E Tests | not_started / in_progress / complete |
> | 4 | Health Check | not_started / in_progress / complete |
> | 5 | Showroom Config | set up / review |
>
> Type a number to work on that item.

**If this is the first visit** (all modules `not_started` and all development fields `not_started`),
append:

> Your showroom is scaffolded and ready for content. Here are some optional helper tools you can
> use — they are not mandatory:
>
> - **Writer helper** — generates module content from your outlines using AI
> - **Reviewer helper** — reviews your `.adoc` files against Red Hat quality standards
> - **Automation helper** — helps build Ansible roles or GitOps configs
>
> Write your content however you prefer. When all workstreams are done, I'll submit to Central.

### Step 3 — Submission gate

**Trigger:** ALL four workstreams are complete:
- All modules have `status: complete`
- `development.automation.status` is `complete`
- `development.e2e.status` is `complete`
- `development.healthCheck.status` is `complete`

**All complete →**
> "All workstreams are complete and ready to submit. Would you like to submit development?"

- **Yes** →
  1. Run `python publishing-house/tools/ph-development.py`. If it fails, STOP and show the error.
  2. Confirm: "Development submitted to Central — workflow advanced to review stage."
- **No** → return to dashboard.

**Not all complete →** do not offer submission. Show the dashboard with outstanding items.

### Step 4 — Workstream selection and dispatch

Based on the user's number selection from the dashboard:

#### Option 1 — Modules

Show the module list with sub-options:

> **Modules**
>
> | # | Module | Status |
> |---|--------|--------|
> | 1 | [title] | not_started / in_progress / complete |
> | 2 | [title] | not_started / in_progress / complete |
> | ... | ... | ... |
>
> **Actions:**
> - Type a module number to **start** it (sets status to `in_progress`)
> - Type `done N` to **mark module N complete**
> - Type `back` to return to the dashboard

**Starting a module:**
  1. Set the module's `status` to `in_progress` in `spec.yaml`
  2. Commit: `git add publishing-house/spec.yaml && git commit -m "feat: start module N — [title]"`

**Marking a module complete:**
  1. Verify the module's `.adoc` file exists in `content/modules/ROOT/pages/`. If not, tell the author.
  2. Update `publishing-house/spec.yaml`: set `status: complete`
  3. Commit and push:
     ```bash
     git add publishing-house/spec.yaml
     git commit -m "feat: mark module N complete — [title]"
     git push
     ```
  4. Close the Jira ticket (best-effort):
     ```bash
     python publishing-house/tools/ph-task-complete.py module-NN
     ```
  5. Return to dashboard.

Module writing and reviewing are handled by optional helper skills (`rhdp-publishing-house:writer-helper`,
`rhdp-publishing-house:reviewer-helper`) — redirect if the author asks to write or review content.

#### Option 2 — Automation

Set `development.automation.status: in_progress` if currently `not_started`, commit, then show:

> **Automation**
>
> | # | Option | Status |
> |---|--------|--------|
> | 1 | GitOps helper | dispatches to gitops-helper skill |
> | 2 | Ansible helper | not yet implemented — build manually |
> | 3 | Mark automation complete | |
> | 4 | Back to dashboard | |

- **1** → dispatch to `rhdp-publishing-house:gitops-helper` skill
- **2** → inform:
  > "The Ansible helper skill is not yet implemented (RHDPCD-110). Please build your Ansible
  > automation manually. Select option 3 when done."
- **3** →
  1. Set `development.automation.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-automation`
  4. Return to dashboard.
- **4** → return to dashboard.

#### Option 3 — E2E Tests

Set `development.e2e.status: in_progress` if currently `not_started`, commit, then show:

> **E2E Tests**
>
> The E2E test helper skill is not yet implemented. Please write your E2E tests manually in
> `qa-automation/`.
>
> | # | Option |
> |---|--------|
> | 1 | Mark E2E tests complete |
> | 2 | Back to dashboard |

- **1** →
  1. Set `development.e2e.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-e2e-tests`
  4. Return to dashboard.
- **2** → return to dashboard.

#### Option 4 — Health Check

Set `development.healthCheck.status: in_progress` if currently `not_started`, commit, then show:

> **Health Check**
>
> The health check helper skill is not yet implemented. Please write your health check playbook
> in `qa-automation/`.
>
> | # | Option |
> |---|--------|
> | 1 | Mark health check complete |
> | 2 | Back to dashboard |

- **1** →
  1. Set `development.healthCheck.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-health-check`
  4. Return to dashboard.
- **2** → return to dashboard.

#### Option 5 — Showroom Config

> | # | Option |
> |---|--------|
> | 1 | Set up showroom (config-helper) |
> | 2 | Review showroom config (config-reviewer) |
> | 3 | Back to dashboard |

- **1** → follow `procedures/config-helper.md`
- **2** → follow `procedures/config-reviewer.md`
- **3** → return to dashboard.

## Rules

- Never tell the author to run any script
- The development skill owns scaffolding, workstream status tracking, and Central submission
- Writing, reviewing, and building automation are optional helper skills
  (`rhdp-publishing-house:writer-helper`, `rhdp-publishing-house:reviewer-helper`,
  `rhdp-publishing-house:automation-helper`) that the author invokes independently
- `config-helper.md` and `config-reviewer.md` are the only procedures this skill dispatches to
- Status transitions: `not_started` → `in_progress` (on selection) → `complete` (on explicit human confirmation only)
- **NEVER mark a workstream complete without the author explicitly saying it is done**
- When adding a new module to `spec.modules`, always set `status: not_started`:
  ```yaml
  - id: module-NN
    title: "Module Title"
    duration_min: 30
    status: not_started
  ```
