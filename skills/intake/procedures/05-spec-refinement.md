# Spec Refinement

This procedure runs after vetting completes (when RCARS results are available).
It incorporates vetting findings and polishes the spec for the approval gate.

**Refine, don't redesign.** The spec reflects the author's vision. You sharpen it; you
do not replace it. If something is genuinely ambiguous, ask — don't fill in details the
author didn't provide.

## Step 1: Read Vetting Findings

Check if any vetting results are available (stored in spec.yaml or a vetting report file).

**If vetting completed with RCARS results**, use them to drive refinement:
- Content gaps identified by RCARS → recommend adding these as differentiators in the
  spec (new sections, expanded module content, explicit positioning)
- High-overlap matches → recommend the spec explicitly articulates how this workshop
  differs from each overlapping item

Present these as **recommendations, not requirements.** The author decides whether to
incorporate them:

> "RCARS identified these content gaps that make your workshop unique. I'd recommend
> highlighting them in the spec — but it's your call."

**If no vetting results are present**, proceed with quality-only refinement.

## Step 2: Review Spec Quality

Re-read all spec artifacts (design.md + module outlines) and review for:

- Missing sections or vague content
- Module outlines with imprecise steps ("navigate to the console and look around")
- Duration estimates that don't match the depth of the outlined content
- Learning objectives that aren't actionable (start with action verbs, not "understand")
- Inconsistent terminology across modules

## Step 3: Propose and Apply Changes

Combine vetting-driven and quality-driven changes into one consolidated list.

### Autonomy Behavior

- **Guided:** Present each proposed change, ask approval before applying.
- **Assisted:** Make all changes, present summary of what changed.
- **Autonomous:** Make changes, present brief summary.

**All autonomy modes require a user checkpoint before marking complete.**

### User Checkpoint (required, all modes)

After applying changes, present a summary and wait for confirmation:

> "I've updated the spec with [N] changes:
>
> **Design spec:**
> - [list key changes to design.md]
>
> **Module outlines:**
> - [list key changes per module]
>
> Review the updated files:
> - `publishing-house/spec/design.md`
> - `publishing-house/spec/modules/`
>
> Let me know if you want to make any changes before we proceed."

**Wait for the user to confirm.** Do not proceed until the user says the spec looks
good, says to proceed, or otherwise confirms.

After confirmation, proceed to `procedures/06-approval-and-submit.md`.
