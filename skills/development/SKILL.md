---
name: rhdp-publishing-house:development
description: This skill should be used when the user asks to "set up showroom", "configure showroom tabs", "create site.yml", "scaffold the showroom structure", "add a tab", "review my showroom config", "check site.yml", "validate ui-config.yml", "development dashboard", "what's next to develop", or "submit to central". Handles showroom scaffolding, config review, workstream status tracking, and submission to Central during the development stage.
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

### Step 2 — In-progress check

Before showing the dashboard, check for any `in_progress` work and ask the author about each one.

**2a. Modules** — For each module with `status: in_progress`, ask:

> "Module N — *[title]* is in progress. Are you done with it?"
> 1. Yes, mark it complete
> 2. No, still working on it

- **1** →
  1. Verify the module's `.adoc` file exists in `content/modules/ROOT/pages/`. If not, warn the author
     but still allow marking complete if they confirm.
  2. Set `status: complete` in spec.yaml
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
- **2** → leave as `in_progress`, move to next.

Ask about all `in_progress` modules sequentially before proceeding.

**2b. Automation** — If `development.automation.status` is `in_progress`, ask:

> "Automation is in progress. Are you done with it?"
> 1. Yes, mark it complete
> 2. No, still working on it

- **1** →
  1. Set `development.automation.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-automation`
- **2** → leave as `in_progress`, move on.

**2c. E2E Tests** — If `development.e2e.status` is `in_progress`, ask:

> "E2E Tests are in progress. Are you done with them?"
> 1. Yes, mark complete
> 2. No, still working on it

- **1** →
  1. Set `development.e2e.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-e2e-tests`
- **2** → leave as `in_progress`, move on.

**2d. Health Check** — If `development.healthCheck.status` is `in_progress`, ask:

> "Health Check is in progress. Are you done with it?"
> 1. Yes, mark complete
> 2. No, still working on it

- **1** →
  1. Set `development.healthCheck.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-health-check`
- **2** → leave as `in_progress`, move on.

After all checks, proceed to Step 3.

### Step 3 — Development Dashboard

Read all workstream statuses from `spec.yaml` (updated after Step 2) and present the dashboard.
This runs every time — it is the central hub of the development phase.

Only show **incomplete** workstreams (not_started or in_progress). Completed workstreams are hidden.

Read from `spec.yaml`:
- Module statuses from `spec.modules[*].status`
- `development.automation.status`
- `development.e2e.status`
- `development.healthCheck.status`

Build the dashboard dynamically based on what is still incomplete:

> **Development Dashboard**
>
> | # | Workstream | Status |
> |---|------------|--------|

Include these rows **only if incomplete**:
- **Modules** — show if any module is not `complete` (display "N of M complete")
- **Automation** — show if `development.automation.status` is not `complete`

Include these rows **only if automation is complete AND the workstream is incomplete**:
- **E2E Tests *(optional)*** — show if `development.e2e.status` is not `complete`
- **Health Check *(optional)*** — show if `development.healthCheck.status` is not `complete`

Always include:
- **Showroom Config** — always shown (set up / review)

Number rows sequentially starting at 1 based on what is shown.

> Type a number to work on that item.

If E2E or Health Check rows are shown, append:
> E2E Tests and Health Check are optional — you can submit without completing them.

**If this is the first visit** (all modules `not_started` and all development fields `not_started`),
append:

> Your showroom is scaffolded and ready for content. Each workstream has optional AI helpers
> available when you select it. Write your content however you prefer — when required workstreams
> are done, I'll submit to Central.

### Step 4 — Submission gate

**Trigger:** Required workstreams are complete:
- All modules have `status: complete`
- `development.automation.status` is `complete`

E2E Tests and Health Check are **not required** for submission.

**Required complete →**
> "Modules and automation are complete. Would you like to submit development?"
> *(If E2E or Health Check are incomplete, add: "E2E Tests and Health Check are still incomplete but are optional.")*

- **Yes** →
  1. Run `python publishing-house/tools/ph-development.py`. If it fails, STOP and show the error.
  2. Confirm: "Development submitted to Central — workflow advanced to review stage."
- **No** → return to dashboard.

**Required not complete →** do not offer submission. Show the dashboard with outstanding items.

### Step 5 — Workstream selection and dispatch

Based on the user's number selection from the dashboard.
The dashboard rows are dynamic — match the user's selection to the workstream shown at that number.

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

**Starting a module (only if `not_started`):**
  1. If the module is already `in_progress` → tell the author it's already in progress, no change needed.
  2. If the module is already `complete` → tell the author it's already complete. Ask if they want to reopen it (set back to `in_progress`).
  3. Otherwise set `status: in_progress` in `spec.yaml`
  4. Commit: `git add publishing-house/spec.yaml && git commit -m "feat: start module N — [title]"`

**Marking a module complete (only if not already `complete`):**
  If already `complete` → tell the author it's already done, return to dashboard.
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

**Optional helpers** — mention these when the author selects a module:
- **Writer helper** (`rhdp-publishing-house:writer-helper`) — generates module content from outlines using AI
- **Reviewer helper** (`rhdp-publishing-house:reviewer-helper`) — reviews `.adoc` files against Red Hat quality standards

Redirect if the author asks to write or review content.

#### Option 2 — Automation

If `development.automation.status` is already `complete`:
> "Automation is already complete. Would you like to reopen it?"
> 1. Reopen (set back to `in_progress`)
> 2. Back to dashboard

Otherwise, set `development.automation.status: in_progress` if currently `not_started`, commit, then
read `project.automation_type` from `spec.yaml` and show the appropriate menu:

**If `automation_type` is `gitops`:**

> **Automation (GitOps)**
>
> | # | Option |
> |---|--------|
> | 1 | GitOps helper (generates Helm + ArgoCD) |
> | 2 | Mark automation complete |
> | 3 | Back to dashboard |

- **1** → dispatch to `rhdp-publishing-house:gitops-helper` skill. When it returns, return to dashboard.
- **2** → set complete, commit, push, close Jira (see below).
- **3** → return to dashboard.

**If `automation_type` is `ansible`:**

> **Automation (Ansible)**
>
> | # | Option |
> |---|--------|
> | 1 | Ansible helper *(not yet implemented — RHDPCD-110)* |
> | 2 | Mark automation complete |
> | 3 | Back to dashboard |

- **1** → inform: "The Ansible helper skill is not yet implemented (RHDPCD-110). Please build your Ansible automation manually. Select option 2 when done."
- **2** → set complete, commit, push, close Jira (see below).
- **3** → return to dashboard.

**If `automation_type` is `both`:**

> **Automation (GitOps + Ansible)**
>
> | # | Option |
> |---|--------|
> | 1 | GitOps helper (generates Helm + ArgoCD) |
> | 2 | Ansible helper *(not yet implemented — RHDPCD-110)* |
> | 3 | Mark automation complete |
> | 4 | Back to dashboard |

- **1** → dispatch to `rhdp-publishing-house:gitops-helper` skill. When it returns, return to dashboard.
- **2** → inform: "The Ansible helper skill is not yet implemented (RHDPCD-110). Please build your Ansible automation manually."
- **3** → set complete, commit, push, close Jira (see below).
- **4** → return to dashboard.

**Marking automation complete (all types):**
  1. Set `development.automation.status: complete` in spec.yaml
  2. Commit and push
  3. Close the Jira ticket: `python publishing-house/tools/ph-task-complete.py write-automation`
  4. Return to dashboard.

#### Option 3 — E2E Tests (only shown when automation is complete)

If `development.e2e.status` is already `complete`:
> "E2E tests are already complete. Would you like to reopen?"
> 1. Reopen (set back to `in_progress`)
> 2. Back to dashboard

Otherwise, set `development.e2e.status: in_progress` if currently `not_started`, commit, then show:

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

#### Option 4 — Health Check (only shown when automation is complete)

If `development.healthCheck.status` is already `complete`:
> "Health check is already complete. Would you like to reopen?"
> 1. Reopen (set back to `in_progress`)
> 2. Back to dashboard

Otherwise, set `development.healthCheck.status: in_progress` if currently `not_started`, commit, then show:

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

#### Option 3 or 5 — Showroom Config

This is option **3** when automation is not complete, or option **5** when it is.

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
