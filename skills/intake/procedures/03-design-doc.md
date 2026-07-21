# Design Doc

This procedure writes or updates the design spec.

## Fresh Intake — Generate From Interview

When called after the interview (no existing design.md):

### Step 1: Generate Design Spec

Generate `design.md` following the design template
(`@rhdp-publishing-house/skills/intake/references/design-template.md`).
Use the template's exact section headings. Fill in placeholders with real content.

### Step 2: Write to Disk

Write the generated spec to `publishing-house/spec/design.md`.

### Step 3: Present Summary

> "I've written the design spec to `publishing-house/spec/design.md`. Here's what it covers:
>
> **[Project Name]** — [one-line goal]
> **Audience:** [audience] | **Duration:** [duration] | **Modules:** [count]
>
> Review or edit the file directly if anything needs changing. When you're ready, I'll
> generate the module outlines."

### Step 4: Wait for Approval

**Do NOT generate module outlines yet.** Wait for explicit approval from the author
before proceeding to `procedures/04-module-outlines.md`.

- **If feedback** → update design.md based on feedback, re-present summary
- **If yes/looks good/proceed** → proceed to `procedures/04-module-outlines.md`

## Re-intake After Rejection — Surgical Update

When called after the rejection handler (01-rejection-handler.md), design.md already
exists. Do NOT regenerate from scratch.

1. Read `publishing-house/spec.yaml` to understand what changed during rejection resolution
2. Read the existing `publishing-house/spec/design.md`
3. Identify every section affected by the spec.yaml changes — this is not a find-and-replace; sections need rewriting so they read coherently with the new values
4. For example, if `cloud_provider` changed from `aws` to `cnv`:
   - Infrastructure Requirements needs rewriting (different platform, different capabilities)
   - Environment section may describe AWS-specific services that no longer apply
   - Cluster sizing assumptions may change
   - Automation approach may differ
5. Present the updated sections to the author for confirmation
6. Apply the changes after confirmation

```bash
git add publishing-house/spec/design.md
git diff --cached --quiet || git commit -m "fix: update design doc to reflect rejection changes" 2>/dev/null || true
```

After confirmation, proceed to `procedures/04-module-outlines.md`.
