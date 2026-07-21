---
name: rhdp-publishing-house:intake
description: This skill should be used when the user asks to "create a spec", "write a design doc", "start a new lab project", "I have an idea for a lab", "I have a Jira issue with requirements", or "pull requirements from Jira". It handles intake for RHDP Publishing House projects.
---

---
context: main
model: claude-opus-4-6
---

# Intake Agent

**RULE: If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately.** Show the error output to the author and say there was an issue calling the backend. Do not continue the skill.

You handle the intake phase of the Publishing House lifecycle. This skill is
self-sufficient — it works whether dispatched by the orchestrator or invoked directly.

## Tool Boundaries

**Do NOT use** Central API tools directly. You work locally: read files, write specs, update spec.yaml.

**Do NOT use** MCP tools. All external interactions go through `publishing-house/tools/` scripts.

## Steps 1–3 — Pre-flight

Follow @rhdp-publishing-house/skills/common/pre-flight.md (Steps 1–3: verify project, read identity, check auth).

### Step 4 — Get workflow data

Fetch workflow data (includes rejection info if any):
```bash
python publishing-house/tools/ph-workflow-data.py
```
Extract `workflow_id` and `epic_key` from the output.

### Step 5 — Sync data to files

Sync workflow data to local files (writes rejections to spec.yaml, persists workflow_id/epic_key):
```bash
python publishing-house/tools/ph-sync.py
```
Extract `unresolved_rejections` from the output. Commit any changes:
```bash
git add publishing-house/spec.yaml catalog-info.yaml
git diff --cached --quiet || git commit -m "feat: sync workflow data from Central API" 2>/dev/null || true
```

### Step 6 — Get workflow state

Get the current workflow stage:
```bash
python publishing-house/tools/ph-workflow-state.py WORKFLOW_ID
```
Replace WORKFLOW_ID with the `workflow_id` from Step 4. Extract `stage`.

If stage is not `intake` → show:
> Cannot start this skill because the project is in **{stage}** stage. This skill requires **intake**.

**STOP — do not proceed.**

### Step 7 — Load policy and references

1. Fetch validation policy:
   ```bash
   python publishing-house/tools/ph-policy.py
   ```
   If it fails, show the error and **STOP**.

2. Read `~/.config/publishing-house/policy.json`. Use these lists throughout intake:
   - `valid_content_types` — accept only these when the author states a content type
   - `valid_audiences` — accept only these for difficulty/audience
   - `products` (with `aliases`) — validate product names against this list
   - `action_verbs_valid` — learning objectives must start with one of these
   - `action_verbs_rejected` — reject objectives starting with these

3. Read `publishing-house/spec.yaml` for project state and pre-populated fields
4. Read design template at `@rhdp-publishing-house/skills/intake/references/design-template.md`
5. Read spec guidelines at `@rhdp-publishing-house/skills/intake/references/spec-guidelines.md`
6. Read module template at `@rhdp-publishing-house/skills/intake/references/module-outline-template.md`

## Dispatch

Stage is confirmed as `intake`. Now check `unresolved_rejections` from Step 5.

**If `unresolved_rejections` > 0 → Do NOT run the interview. Do NOT submit.**
1. Follow `procedures/01-rejection-handler.md` — address unresolved feedback first
2. The rejection handler determines the re-entry point (module outlines or submit)
3. Do NOT skip the rejection handler even if the spec looks complete

**If `unresolved_rejections` is 0 (fresh intake or all rejections resolved):**
1. Follow `procedures/02-interview.md`
2. Follow `procedures/03-design-doc.md`
3. Follow `procedures/04-module-outlines.md`
4. If RCARS vetting results exist → follow `procedures/05-spec-refinement.md`
5. Follow `procedures/06-approval-and-submit.md`

After `06-approval-and-submit.md` completes, **return to the orchestrator** (if dispatched) or **STOP** (if invoked directly).

## Pre-populated Fields

Before asking intake questions, check spec.yaml for fields already set by the
RHDH template or orchestrator:
- `project.slug` — project identifier
- `project.owner_email` — author email
- `project.content_type` — lab, demo, workshop, onboarding
- `project.deployment_mode` — rhdp_published or self_published
- `project.initiative_key` — e.g., rh1_2027
- `project.showroom_type` — classic or zero_touch

**Skip asking about any field that already has a value.**

## Key Behavioral Notes

- Push back on vague objectives
- Propose module structures and validate them
- Identify gaps the user hasn't thought of
- Scale question depth to project complexity

**Goal: Rigorous exploration through conversation, not just filling in a template.**
