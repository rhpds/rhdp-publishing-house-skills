# Rejection Handler

This procedure runs when intake detects rejection data from `ph-workflow.py`.
The project was previously submitted but rejected at Content Review or Infra Review.

## Step 1: Show Rejection Context

Present the rejection to the author:

> **This project was sent back for revisions.**
>
> **Rejected at:** {rejection_source} (Content Review or Infra Review)
> **Rejected by:** {rejection_by}
>
> **Reasons:**
> {rejection_reasons}

## Step 2: Review Current State

Read the existing spec artifacts — they are still in place from the previous intake:
- `publishing-house/spec/design.md`
- `publishing-house/spec/modules/` (all module outlines)
- `publishing-house/spec.yaml`

Present a brief summary of the current spec so the author has context.

## Step 3: Guided Resolution

Walk through each rejection reason one at a time:

1. Show the reason
2. Show the relevant section of design.md or spec.yaml that relates to it
3. Ask the author how they want to address it
4. Apply the fix (update design.md, spec.yaml, or module outlines as needed)

**Do NOT re-run the full interview.** The spec already exists. Only fix what was rejected.

## Step 4: Determine Re-entry Point

After all rejection reasons are addressed, ask the author:

> "Are you happy with these changes and ready to resubmit for review?"

If the author confirms, determine what changed and where to rejoin:

- **If design.md structure changed** (modules added/removed, learning objectives rewritten, scope changed):
  → Continue from `procedures/04-module-outlines.md` to surgically update affected module outline sections, then `procedures/06-approval-and-submit.md`

- **If only spec.yaml changed** (infra settings, audience, duration) or design.md wording changed (no structural impact):
  → Continue from `procedures/06-approval-and-submit.md` to resubmit

The SKILL.md dispatch section handles this routing.
