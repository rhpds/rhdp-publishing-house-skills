---
name: rhdp-publishing-house:development
description: This skill should be used when the user asks to "write a module", "draft content", "start writing", "edit my content", "review the modules", "build automation", "write the Ansible roles", "set up GitOps", "module N is done", "mark it complete", "review again", "what's next to develop", "set up showroom", "configure showroom tabs", "create site.yml", "scaffold the showroom structure", "add a tab", "review my showroom config", "check site.yml", or "validate ui-config.yml". Handles writing, editing, automation, scaffolding, config review, module completion, and re-review during the development stage.
context: main
---

# Development Agent

**RULE: If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately.** Show the error output to the author and say there was an issue calling the backend. Do not continue the skill.

You handle the development phase of the Publishing House lifecycle. This skill is
self-sufficient — it works whether dispatched by the orchestrator or invoked directly.

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

### Step 1 — Readiness check (runs FIRST, before anything else)

**Trigger:** All modules have `status: complete` in spec.yaml.
**Skip if** any module is `not_started` or `in_progress` — proceed directly to Step 2.

Run these checks:

1. `content/modules/ROOT/pages/index.adoc` exists
2. `content/modules/ROOT/pages/conclusion.adoc` exists
3. `content/modules/ROOT/nav.adoc` exists
4. Every module outline in `publishing-house/spec/modules/` has a matching `.adoc` page in `content/modules/ROOT/pages/`
5. No placeholder text (`TODO`, `FIXME`, `TBD`, `[placeholder]`) in any `.adoc` page
6. All learning objectives from `spec.learning_objectives` are referenced in `conclusion.adoc`

**All checks pass →**
> "All content is complete and ready to submit. Would you like to submit development, or is there something else you'd like to work on?"

- **Yes** → run `python publishing-house/tools/ph-development.py`. If it fails, STOP and show the error.
- **No** → proceed to Step 3 dispatch.

**Any check fails →** list what's missing and proceed to Step 2.

### Step 2 — Scaffold check gate

Follow `procedures/config-reviewer.md` automatically against the project's content directory.

- **PASS** → proceed to Step 2b
- **FAIL** → report the specific issues to the author, then ask:
  > "The showroom scaffold has issues that need to be resolved first. Would you like me to help fix them, or will you handle it?"
  - "help me" → follow `procedures/config-helper.md` to fix the scaffold, then proceed to Step 2b
  - "I'll handle it" → STOP. Do not proceed until the author says the scaffold is ready.

### Step 2b — Module status validation gate

Read `spec.yaml` and check module statuses.

- **Any module is `in_progress`?** → warn the author:
  > "Module N is currently marked in_progress. Would you like to continue it, or mark it complete first?"
  Wait for the author's response before dispatching.
- **All modules are `complete` AND user request is "write"?** → suggest editing instead:
  > "All modules are already complete. Did you mean to edit or review the content instead?"
  Wait for confirmation.
- **Otherwise** → proceed to Step 2c.

### Step 2c — Development mode selection (first time only)

**Skip this step if** any module has `status: in_progress` or `status: complete` — the author has already started development.

**Trigger:** All modules are `not_started` AND the author's request involves writing content ("write", "start writing", "write all", "write module N").

> **How would you like to develop your content?**
>
> 1. **Use PH Writer** — I'll generate modules from your outlines, run the reviewer, track status in spec.yaml, and submit to Central when done. Fully managed.
>
> 2. **Write on your own** — Please write your `.adoc` files yourself or use your own tools. A few things to keep in mind:
>    - Please update each module's `status` in `publishing-house/spec.yaml` manually (`not_started` → `in_progress` → `complete`)
>    - Please run backend scripts manually to keep Central in sync — PH will not run them for you
>
> Which approach would you prefer?

**Wait for the author's response.**

- **Option 1** → proceed to Step 3 dispatch (follow `procedures/writer.md`)
- **Option 2** →
  > "Understood — you're in charge of writing. Please remember to update module statuses in `publishing-house/spec.yaml` as you go, and run the backend scripts when you're ready to submit. If you need help later, just ask."
  >
  > **STOP.**

### Step 3 — Dispatch

Based on what the user asked for:

- **"write module N"** / **"start writing"** / **"write all"** → follow `procedures/writer.md`
- **"edit module N"** / **"review content"** / **"technical edit"** → follow `procedures/editor.md`
- **"build automation"** / **"write the Ansible roles"** / **"set up GitOps"** → follow `procedures/automation.md`
- **"set up showroom"** / **"configure tabs"** / **"scaffold"** / **"add a tab"** → follow `procedures/config-helper.md`
- **"review config"** / **"check site.yml"** / **"validate config"** → follow `procedures/config-reviewer.md`
- **"module N is done"** / **"mark module N complete"** / **"it's done"** / **"looks good"** → follow the completion flow in `procedures/writer.md` Step 5d (update spec.yaml status to `complete`)
- **"review again"** / **"re-review"** / **"check it again"** → follow `procedures/writer.md` Step 5c-retry (re-run reviewer on the current `.adoc` file)
- **No specific request** / **"what's next"** → show development dashboard:

### Development Dashboard

Present the current state:

> **Development Status**
>
> **Modules:**
> [For each module outline, check if a corresponding .adoc exists in content/modules/ROOT/pages/]
> - Module 1: [title] — [written / not started]
> - Module 2: [title] — [written / not started]
>
> **Automation:** [done / not started]
>
> What would you like to work on?

## Rules

- Never tell the author to run any script
- The development skill dispatches to procedures but does not own workflow advancement
- Each procedure handles its own commit and push
- When adding a new module to `spec.modules`, always set `status: not_started`:
  ```yaml
  - id: module-NN
    title: "Module Title"
    duration_min: 30
    status: not_started
  ```
