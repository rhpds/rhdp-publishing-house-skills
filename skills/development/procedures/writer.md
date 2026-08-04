# Writer

You write Showroom AsciiDoc content by spawning the `rhdp-publishing-house:module-writing-helper` agent
with context from the project's spec files. You do NOT write AsciiDoc directly.

See @rhdp-publishing-house/skills/development/references/writing-standards.md for writing standards.

## CRITICAL CONSTRAINT — Content Only, No Scaffolding

You write `.adoc` module files into an already-scaffolded showroom repo.
You MUST NOT create or modify scaffold files (`site.yml`, `ui-config.yml`, `antora.yml`, directory structure).
Scaffolding is handled by `rhdp-publishing-house:config-helper` (RHDPCD-172) or manually before development begins.

## Step 1: Determine Which Module to Write

Check what the user requested:

- If user said "write module N" → write that specific module
- If user said "write all" or "start writing" → write all pending modules sequentially
- **Never write modules in parallel.** Each module depends on the previous one for story continuity.

## Step 2: Read Project Context

Read three files from the author's project repo:

1. **`publishing-house/spec.yaml`** — machine-readable metadata: environment (ocp_version, topology, cloud_provider), module list, audience, duration
2. **`publishing-house/spec/design.md`** — human-readable narrative: overview, audience, prerequisites, products, business scenario
3. **`publishing-house/spec/modules/module-NN-*.md`** — detailed step-by-step outline for the target module

Build a combined context object from all three sources.

## Step 3: Check Module Status (Sequential Enforcement)

Before spawning any agent, check `spec.yaml` module statuses:

- `not_started` → eligible to write; set to `in_progress` when starting
- `in_progress` → resume this module (started but not finished)
- `complete` → skip; move to next module

**Sequential rule:** Module N CANNOT start until modules 1 through N-1 are ALL `complete`.

Present a plan before spawning:
> "Here's what I'll write for module N: [summary of outline]. Ready to proceed?"

Wait for user approval — never auto-generate.

## Step 4: Spawn rhdp-publishing-house:module-writing-helper Agent

For each module, spawn the agent via Task tool:

    Task tool:
      subagent_type: rhdp-publishing-house:module-writing-helper
      prompt: |
        TARGET_FILE: content/modules/ROOT/pages/<module-filename>.adoc
        FILE_TYPE: module
        FULL_SPEC: <JSON from spec.yaml + design.md + module outline>
        LAB_TYPE: <ocp|rhel|vm|ai>
        CONTENT_TYPE: <workshop|demo>
        REPO_PATH: <absolute repo path>

One agent per module, run sequentially.

## Step 5: Post-Generation Verification

After the writing agent finishes:

1. Verify the generated file exists in `content/modules/ROOT/pages/`
2. Check that `content/modules/ROOT/nav.adoc` includes the new module
3. Scan for placeholders and open items:
   - Image references (`image::`) where the image file doesn't exist in `content/modules/ROOT/assets/images/`
   - Placeholder text like `TODO`, `FIXME`, `[placeholder]`, `TBD`
   - Diagram references without corresponding files
4. Collect all open items into a list

## Step 5b: Auto-Run Reviewer

**Do NOT skip this step. The reviewer runs automatically after every write — the author does not need to ask for it.**

Spawn the reviewer agent:

    Agent tool:
      subagent_type: rhdp-publishing-house:module-reviewer
      prompt: |
        MODULE_FILE: <absolute path to the just-written .adoc file>
        CONTENT_TYPE: <workshop|demo>
        LAB_TYPE: <ocp|rhel|vm|ai>
        SHARED_CONTEXT: <JSON with module_order, defined_attributes, first_use_map, is_first_module, is_conclusion>
        REPO_PATH: <absolute repo path>

Wait for the reviewer to complete. Extract the dimension scores and findings.

## Step 5c: Present Results and STOP for Human Review

Present a summary to the author:

> **Module N — Writing Complete, Awaiting Your Review**
>
> **Reviewer Score:** [overall score or per-dimension breakdown]
>
> **Findings:** [count] issues found
> [List HIGH and CRITICAL findings]
>
> **Open Items:**
> - [List any missing images, placeholders, TODOs from Step 5]
>
> **Please review the generated file:**
> `content/modules/ROOT/pages/[filename].adoc`
>
> Open it, read through the content, and check that it matches what you expect.
> The AI handled ~80% — your review covers the rest: accuracy, tone, missing context, and any items above.
>
> When you've reviewed and addressed these items, say **"module N is done"** and I'll mark it complete.

**HARD STOP HERE.** Do NOT mark the module complete. Do NOT proceed to the next module.
Wait for the author to explicitly say the module is done.

## Step 5d: Mark Complete (only after human approval)

When the author says "module N is done" / "it's done" / "mark it complete" / "looks good":

1. Update `publishing-house/spec.yaml`: change `status: in_progress` to `status: complete` for this module
2. Commit:
   ```bash
   git add publishing-house/spec.yaml
   git commit -m "feat: mark module N complete — [title]"
   ```
3. Confirm:
   > "Module N marked complete. [Next module available / All modules complete.]"

## Step 6: Commit the Written Content

Commit the content file and nav update immediately after writing (before review):

    git add content/
    git commit -m "feat: write module N — [title]"

Note: The completion status commit (Step 5d) is SEPARATE from the content commit.

## What You Do NOT Do

- **NEVER write AsciiDoc files directly** — always spawn rhdp-publishing-house:module-writing-helper
- **NEVER create or modify scaffold files** (`site.yml`, `ui-config.yml`, `antora.yml`, `nav.adoc` structure)
- **NEVER write modules in parallel**
- **NEVER mark a module complete without human approval** — present findings and open items, then wait
- **NEVER skip the reviewer** — it runs automatically after every write
