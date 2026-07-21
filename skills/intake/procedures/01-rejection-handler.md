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
2. Determine what it affects — it may be a spec.yaml field, a design.md section, a module outline, or a combination
3. Show the relevant section(s) from the affected file(s)
4. Discuss with the author how to address it
5. Apply the agreed fix to the appropriate file(s):
   - **Spec change** (e.g. cloud_provider, sizing, products) → update spec.yaml
   - **Content change** (e.g. "learning objectives are too vague", "missing prerequisites") → update the module outline or design.md directly
   - **Both** (e.g. "wrong infrastructure and module steps don't match") → update both
6. Move to the next reason

Do NOT commit yet — all changes are applied first.

## Step 5: Confirm Resolution

After all reasons have been addressed, summarize what changed and where:

> **Here's what was changed:**
> 1. {reason text} → {what was changed and in which file}
> 2. {reason text} → {what was changed and in which file}
>
> **Are you happy with these changes?**

- **If no** or has more changes → go back to the relevant reason in Step 4
- **If yes** → commit and continue

## Step 6: Cascade Updates

Changes in one file may require updates to others for consistency. Check and update as needed:

- **If spec.yaml changed** → follow `procedures/03-design-doc.md` (Re-intake After Rejection section) to update the design doc, then follow `procedures/04-module-outlines.md` (Re-intake After Rejection section) to update affected module outlines
- **If design.md changed** (directly or via spec cascade) → follow `procedures/04-module-outlines.md` (Re-intake After Rejection section) to update affected module outlines
- **If only module outlines changed** → no cascade needed

Commit all changes:
```bash
git add publishing-house/spec.yaml publishing-house/spec/
git diff --cached --quiet || git commit -m "fix: address rejection feedback" 2>/dev/null || true
```

## Step 7: Mark Resolved and Submit

Mark every addressed rejection reason as `resolved: true` in spec.yaml.

```bash
git add publishing-house/spec.yaml
git diff --cached --quiet || git commit -m "fix: mark rejection reasons as resolved" 2>/dev/null || true
```

Follow `procedures/06-approval-and-submit.md` to resubmit. That procedure asks the author to confirm before submitting.
