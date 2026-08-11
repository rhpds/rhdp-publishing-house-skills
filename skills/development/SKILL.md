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

### Step 1 — Readiness check (runs FIRST, before anything else)

**Trigger:** All modules have `status: complete` in spec.yaml.
**Skip if** any module is `not_started` or `in_progress` — proceed directly to Step 2.

Run these checks against the current state of the repo (do not rely on any previously-saved
"complete" flag — recompute this every time):

1. `content/modules/ROOT/pages/index.adoc` exists
2. `content/modules/ROOT/pages/conclusion.adoc` exists
3. `content/modules/ROOT/nav.adoc` exists
4. Every module outline in `publishing-house/spec/modules/` has a matching `.adoc` page in `content/modules/ROOT/pages/`
5. No placeholder text (`TODO`, `FIXME`, `TBD`, `[placeholder]`) in any `.adoc` page
6. All learning objectives from `spec.learning_objectives` are referenced in `conclusion.adoc`

**All checks pass →**
> "All content is complete and ready to submit. Would you like to submit development, or is there something else you'd like to work on?"

- **Yes** →
  1. Run `python publishing-house/tools/ph-development.py`. If it fails, STOP and show the error.
  2. Confirm: "Showroom content finalized and submitted to Central — workflow advanced to review stage."
- **No** → proceed to Step 2 dispatch.

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
- **Otherwise** → proceed to Step 3.

### Step 3 — Module status management

Handle status transitions in `publishing-house/spec.yaml` directly. This is the only place module
status is authoritatively tracked — it works the same whether the author wrote content by hand,
used `rhdp-publishing-house:writer-helper`, or something else entirely.

- **"start module N"** / **"module N in progress"** →
  1. Set the module's `status: not_started` to `status: in_progress` in `spec.yaml`
  2. Commit: `git add publishing-house/spec.yaml && git commit -m "feat: start module N — [title]"`

- **"module N is done"** / **"mark module N complete"** / **"it's done"** / **"looks good"** →
  1. Verify the module's `.adoc` file exists in `content/modules/ROOT/pages/`. If it doesn't, tell the
     author and do not mark complete.
  2. Update `publishing-house/spec.yaml`: set `status: complete` for that module (from whatever its
     current status was).
  3. Commit and push:
     ```bash
     git add publishing-house/spec.yaml
     git commit -m "feat: mark module N complete — [title]"
     git push
     ```
  4. Close the module's Jira ticket (best-effort):
     ```bash
     python publishing-house/tools/ph-module-complete.py module-NN
     ```
     If there is no epic (self-published mode) or no matching ticket, the script exits cleanly. Do not stop on failure.
  5. Confirm:
     > "Module N marked complete and pushed. [Next module available / All modules complete.]"

  This works standalone too — if the author returns in a new session and says "module N is done"
  without any prior write activity this session, still verify the `.adoc` file exists and mark complete.

### Step 4 — Dispatch

Based on what the user asked for:

- **"set up showroom"** / **"configure tabs"** / **"scaffold"** / **"add a tab"** → follow `procedures/config-helper.md`
- **"review config"** / **"check site.yml"** / **"validate config"** → follow `procedures/config-reviewer.md`
- **"module N is done"** / **"start module N"** / other status phrases → handled by Step 3 above
- **"write a module"** / **"draft content"** / **"start writing"** / **"edit module N"** / **"review content"** / **"technical edit"** / **"build automation"** / **"write the Ansible roles"** / **"set up GitOps"** → redirect:
  > "That's handled by an optional helper skill now, not by me directly. Ask me to run `rhdp-publishing-house:writer-helper`, `rhdp-publishing-house:reviewer-helper`, or `rhdp-publishing-house:automation-helper` — or invoke it yourself. I'll keep tracking module status and handling submission to Central whenever you're ready."
- **No specific request** / **"what's next"** → show development dashboard:

### Development Dashboard

Present the current state, reading module status directly from `spec.yaml` (not file presence):

> **Development Status**
>
> **Modules:**
> [For each module in spec.yaml, show its title and status]
> - Module 1: [title] — [not_started / in_progress / complete]
> - Module 2: [title] — [not_started / in_progress / complete]
>
> What would you like to work on?

**If the scaffold gate (Step 2) just passed and every module is still `not_started`** (i.e. this is
the first time the repo is ready for content), append the optional-helpers blurb before the "what
would you like to work on" line:

> Your showroom is scaffolded and ready for content. Here are some optional helper tools you can
> use — they are not mandatory:
>
> - **Writer helper** — generates module content from your outlines using AI
> - **Reviewer helper** — reviews your `.adoc` files against Red Hat quality standards
> - **Automation helper** — helps build Ansible roles or GitOps configs
>
> Write your content however you prefer. When all modules are done, update each module's status to
> `complete` and say "submit to central" — I'll handle the rest.

## Rules

- Never tell the author to run any script
- The development skill owns scaffolding, module status tracking, and Central submission. It does
  not write, review, or build automation itself — those are optional helper skills
  (`rhdp-publishing-house:writer-helper`, `rhdp-publishing-house:reviewer-helper`,
  `rhdp-publishing-house:automation-helper`) that the author invokes independently
- `config-helper.md` and `config-reviewer.md` are the only procedures this skill dispatches to —
  each handles its own commit and push
- When adding a new module to `spec.modules`, always set `status: not_started`:
  ```yaml
  - id: module-NN
    title: "Module Title"
    duration_min: 30
    status: not_started
  ```
