# Module Outlines

This procedure generates or updates module outlines from design.md.

## Fresh Intake — Generate All Outlines

Module outlines MUST be generated from the written design.md — not from conversation
context. Use the Agent tool to spawn a fresh subagent.

Spawn with a prompt like:

```
Read the design spec at <project_root>/publishing-house/spec/design.md.

Read the module outline template at @rhdp-publishing-house/skills/intake/references/module-outline-template.md.

For each module in the Module Map table, generate one outline file:
- Output directory: <project_root>/publishing-house/spec/modules/
- Naming: module-01-<short-title>.md, module-02-<short-title>.md, etc.
- Follow the template structure exactly.
- Reflect what design.md says — do not invent content not in the spec.
```

## Re-intake After Rejection — Surgical Updates

When called after the rejection handler (01-rejection-handler.md), module outlines
already exist. Do NOT regenerate from scratch.

1. Read the updated design.md to identify what changed
2. Read the existing module outlines in `publishing-house/spec/modules/`
3. For each affected module, update only the sections that changed:
   - If learning objectives changed → update the objectives section in that module
   - If a module was added → generate a new outline file for it
   - If a module was removed → delete the outline file
   - If scope/content changed for a module → update the relevant sections
4. Leave untouched modules as-is

After outlines are written/updated, proceed to the next procedure in the flow.
