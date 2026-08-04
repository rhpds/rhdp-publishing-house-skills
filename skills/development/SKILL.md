---
name: rhdp-publishing-house:development
description: This skill should be used when the user asks to "write a module", "draft content", "start writing", "edit my content", "review the modules", "build automation", "create the catalog", or "what's next to develop". Handles writing, editing, and automation during the development stage.
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

Based on what the user asked for:

- **"write module N"** / **"start writing"** / **"write all"** → follow `procedures/writer.md`
- **"edit module N"** / **"review content"** / **"technical edit"** → follow `procedures/editor.md`
- **"build automation"** / **"create the catalog"** / **"write the AgnosticV config"** → follow `procedures/automation.md`
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
