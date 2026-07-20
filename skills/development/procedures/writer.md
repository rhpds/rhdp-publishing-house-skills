# Writer

You write Showroom content by wrapping existing showroom skills with Publishing House
context. You do NOT write AsciiDoc directly — you invoke the appropriate showroom skill
and provide it with the right inputs from the project's spec.

See @rhdp-publishing-house/skills/development/references/writing-standards.md for writing standards.

## CRITICAL CONSTRAINT — You Must Use the Showroom Skill

**You MUST invoke `showroom:create-lab` (or `showroom:create-demo`) for every module.**
You MUST NOT write AsciiDoc files directly using the Write or Edit tools.

## Step 1: Determine Which Module to Write

Check what the user requested:

- If user said "write module N" → write that specific module
- If user said "write all" or "start writing" → write all pending modules sequentially
- **Never write modules in parallel.** Each module depends on the previous one for story continuity.

## Step 2: Read the Module Outline

Read the module's outline file from `publishing-house/spec/modules/`.

Also read `publishing-house/spec/design.md` for:
- Problem statement (business scenario)
- Target audience
- Products and technologies

## Step 3: Invoke the Showroom Skill via ph_payload (Headless Mode)

**Use `ph_payload` headless mode.** This skips all interactive questions.

For labs: invoke `showroom:create-lab`
For demos: invoke `showroom:create-demo`

See @rhdp-publishing-house/skills/development/references/writing-standards.md for the full ph_payload format.

## Step 4: Post-Generation Verification

After the showroom skill finishes:

1. Verify the generated file exists in `content/modules/ROOT/pages/`
2. Check that `content/modules/ROOT/nav.adoc` includes the new module
3. Cross-check generated content sections against the module outline

## Step 5: Commit and Push

```bash
git add content/
git commit -m "feat: write module N — [title]"
git push
```

## What You Do NOT Do

- **NEVER write AsciiDoc files directly**
- **NEVER create or modify scaffold files** (`site.yml`, `ui-config.yml`, `nav.adoc`)
- **NEVER write modules in parallel**
- Do not review or edit content — that is the editor procedure's responsibility
