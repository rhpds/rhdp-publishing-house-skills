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

## Step 4: Address Each Rejection Reason

Loop through each **unresolved** reason one at a time.

**Do NOT re-run the full interview.** The spec already exists. Only fix what was rejected.

**Do NOT mark reasons as resolved yet.** That happens after everything is updated and confirmed.

For each unresolved reason:

1. Show the reason text
2. Show the relevant section of spec.yaml that relates to it
3. Discuss with the author how to address it — this is a conversation, not a find-and-replace
4. Apply the agreed fix to spec.yaml
5. Move to the next reason

Do NOT commit yet — all spec.yaml changes are applied in memory first.

## Step 5: Confirm Resolution

After all reasons have been addressed, summarize the changes:

> **Here's what changed in the spec:**
> 1. {reason text} → {what was changed}
> 2. {reason text} → {what was changed}
>
> **Are you happy with these changes?**

- **If no** or has more changes → go back to the relevant reason in Step 4
- **If yes** → commit and continue:

```bash
git add publishing-house/spec.yaml
git diff --cached --quiet || git commit -m "fix: address rejection feedback in spec" 2>/dev/null || true
```

## Step 6: Update Design Doc

The design doc must reflect the spec.yaml changes. This is not a find-and-replace — the design doc needs to be reworked so it reads coherently with the new values.

Follow `procedures/03-design-doc.md` (Re-intake After Rejection section) to update the design doc.

## Step 7: Update Module Outlines

Module outlines may reference infrastructure, products, commands, or assumptions that changed.

Follow `procedures/04-module-outlines.md` (Re-intake After Rejection section) to update affected module outlines.

## Step 8: Mark Resolved and Submit

Mark every addressed rejection reason as `resolved: true` in spec.yaml.

```bash
git add publishing-house/spec.yaml
git diff --cached --quiet || git commit -m "fix: mark rejection reasons as resolved" 2>/dev/null || true
```

Follow `procedures/06-approval-and-submit.md` to resubmit. That procedure asks the author to confirm before submitting.
