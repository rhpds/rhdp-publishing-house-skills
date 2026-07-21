# Rejection Handler

This procedure runs when `ph-sync.py` reports `unresolved_rejections` > 0.
The project was previously submitted but rejected at Content Review or Infra Review.
There may be multiple rejection rounds — each is stored separately in spec.yaml.

## Step 1: Read Rejections

Read `publishing-house/spec.yaml` and look at `approval_checklist.content.rejections`
and `approval_checklist.infra.rejections`. Each entry has:

```yaml
- rejection_id: "abc-123"
  reviewer: "John Doe"
  timestamp: "2026-07-20T14:30:00Z"
  reasons:
    - id: "r1"
      text: "Module 3 learning objectives are too vague"
      resolved: false
    - id: "r2"
      text: "Missing prerequisites section"
      resolved: true
```

Collect all reasons where `resolved: false` across all rejections in both sections.
If none are unresolved, **skip this procedure entirely** — return to the caller.

## Step 2: Show Rejection Context

Present each unresolved rejection to the author, grouped by review stage:

> **This project has unresolved review feedback.**
>
> **Content Review** (rejected by {reviewer}, {timestamp}):
> 1. {reason text} — **unresolved**
> 2. {reason text} — **unresolved**
>
> **Infra Review** (rejected by {reviewer}, {timestamp}):
> 1. {reason text} — **unresolved**

Only show rejections that have at least one unresolved reason. Skip fully-resolved ones.

## Step 3: Review Current State

Read the existing spec artifacts for context:
- `publishing-house/spec/design.md`
- `publishing-house/spec/modules/` (all module outlines)
- `publishing-house/spec.yaml`

Present a brief summary so the author knows what they're working with.

## Step 4: Guided Resolution

Walk through each **unresolved** reason one at a time:

1. Show the reason text
2. Show the relevant section of design.md or spec.yaml that relates to it
3. Ask the author how they want to address it
4. Apply the fix (update design.md, spec.yaml, or module outlines as needed)
5. Mark the reason as `resolved: true` in spec.yaml
6. Commit the change

To mark a reason resolved, update spec.yaml directly — find the matching rejection_id
and reason id, set `resolved: true`.

**Do NOT re-run the full interview.** The spec already exists. Only fix what was rejected.

## Step 5: Commit and Determine Re-entry Point

After all unresolved reasons are addressed:

```bash
git add publishing-house/spec.yaml publishing-house/spec/
git diff --cached --quiet || git commit -m "fix: address review rejection feedback" 2>/dev/null || true
```

Ask the author:

> "All rejection feedback has been addressed. Ready to resubmit for review?"

If confirmed, determine what changed and where to rejoin:

- **If design.md structure changed** (modules added/removed, learning objectives rewritten, scope changed):
  → Continue from `procedures/04-module-outlines.md` to update affected outlines, then `procedures/06-approval-and-submit.md`

- **If only spec.yaml changed** (infra settings, audience, duration) or design.md wording changed (no structural impact):
  → Continue from `procedures/06-approval-and-submit.md` to resubmit
