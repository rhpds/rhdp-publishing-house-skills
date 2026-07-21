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

Walk through each **unresolved** reason one at a time.

**Do NOT re-run the full interview.** The spec already exists. Only fix what was rejected.

For each unresolved reason:

1. Show the reason text
2. Show the relevant section of spec.yaml, design.md, or module outlines that relates to it
3. Discuss with the author how to address it — this is a conversation, not a find-and-replace
4. Apply the agreed fix to spec.yaml
5. Mark the reason as `resolved: true` in spec.yaml (find the matching rejection_id and reason id)

To mark a reason resolved, update spec.yaml directly — set `resolved: true` on the matching reason.

Commit after each reason:
```bash
git add publishing-house/spec.yaml
git diff --cached --quiet || git commit -m "fix: resolve rejection reason — {brief description}" 2>/dev/null || true
```

## Step 5: Update Design Doc

After all reasons are resolved in spec.yaml, the design doc must reflect the changes.

**Do NOT do a find-and-replace.** Read the full design doc, understand what changed in spec.yaml, and rewrite the affected sections so the design doc is coherent with the new spec values.

For example, if `cloud_provider` changed from `aws` to `cnv`:
- The Infrastructure Requirements section needs rewriting (not just swapping a word)
- The Environment section may describe AWS-specific services that no longer apply
- Cluster sizing assumptions may change
- Automation approach may differ

1. Read `publishing-house/spec/design.md`
2. Identify every section affected by the spec.yaml changes
3. Rewrite those sections to be consistent with the updated spec
4. Present the updated design doc to the author for confirmation
5. Apply the changes after confirmation

```bash
git add publishing-house/spec/design.md
git diff --cached --quiet || git commit -m "fix: update design doc to reflect rejection changes" 2>/dev/null || true
```

## Step 6: Update Module Outlines

Module outlines may reference infrastructure, products, or assumptions that changed.

1. Read each module outline in `publishing-house/spec/modules/`
2. For each module, check if any content references values that changed (infrastructure, cloud provider, products, commands, environment details)
3. If a module is affected, rewrite the affected sections — steps, infrastructure notes, commands, and context paragraphs should reflect the new reality
4. Present each updated module to the author for confirmation before writing

```bash
git add publishing-house/spec/modules/
git diff --cached --quiet || git commit -m "fix: update module outlines to reflect rejection changes" 2>/dev/null || true
```

If no modules are affected, skip this step and say so.

## Step 7: Final Confirmation and Resubmit

After spec.yaml, design.md, and module outlines are all consistent:

Ask the author:

> "All rejection feedback has been addressed and the design doc and modules have been updated. Ready to resubmit for review?"

If confirmed → continue from `procedures/06-approval-and-submit.md` to resubmit.
