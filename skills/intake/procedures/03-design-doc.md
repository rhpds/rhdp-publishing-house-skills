# Design Doc

Phase 2 of the intake flow. Produce a complete, validated design.md.

## Generate the Design Doc

Read `publishing-house/spec/design.md` from the project repo. This is the template — it has
placeholder sections that you will fill in using what you learned in Phase 1 (discovery).

Read `publishing-house/spec.yaml` inline comments to understand valid values for constrained
fields (e.g., `# cnv | aws | azure` for cloud_provider). Use these as guidance when filling
in the corresponding design.md sections.

For each section in design.md:
1. Replace the placeholder text with real content from the discovery conversation
2. Keep the section heading exactly as-is — the validation engine checks for these
3. If you don't have enough information for a section, leave a clear note: "TBD — [what's needed]"

**Propose module structure.** Based on the discovery conversation, propose a Module Map table
with titles and estimated durations. Explain why you structured it this way. The author may
adjust — that's expected.

## Present for Review

Present the design doc to the author — the design.md content only, NOT spec.yaml:

> "Here's the design doc I've drafted. Review it and let me know if anything needs changing.
> You can also edit `publishing-house/spec/design.md` directly in your editor."

**Wait for explicit approval.** Do NOT generate module outlines until the author says the
design looks good.

- **If feedback** → update design.md, re-present
- **If "I already filled this out"** → read the existing design.md, validate it has content
  (not just placeholders), and proceed
- **If approved** → write and continue

## Re-intake After Rejection

When called after the rejection handler (01-rejection-handler.md), design.md already exists.
Do NOT regenerate from scratch.

1. Read `publishing-house/spec.yaml` to understand what changed during rejection resolution
2. Read the existing `publishing-house/spec/design.md`
3. Identify sections affected by the changes — rewrite them coherently
4. Present the updated sections to the author for confirmation

## Write Point

Write design.md and update corresponding spec.yaml fields:

```bash
git add publishing-house/spec/design.md publishing-house/spec.yaml
git diff --cached --quiet || git commit -m "feat: phase 2 — design doc generated" 2>/dev/null || true
```

## Inline Structure Check

After writing design.md, run a quick validation against the spec guidelines
(`@rhdp-publishing-house/skills/intake/references/spec-guidelines.md`):

- All required sections present (11 sections + descriptive H1 title)
- Learning objectives use valid action verbs (from policy)
- No unfilled template placeholders (`[placeholder text]` markers)
- Module durations in 10-60 minute range
- Module Map table exists with at least one row

**This check is non-blocking.** Show the results. If there are issues, help the author fix
them before proceeding. But do not gate the flow — the hard gate is at submission (Phase 6).

> "Quick structure check on your design doc: [results]. Want to fix these now, or proceed?"

Proceed to Phase 3 (RCARS vetting).
