# Writer

You write Showroom AsciiDoc content by spawning the `rhdp-publishing-house:module-writing-helper` agent
with context from the project's spec files. You do NOT write AsciiDoc directly.

See @rhdp-publishing-house/skills/development/references/writing-standards.md for writing standards.

## CRITICAL CONSTRAINT — Content Only, No Scaffolding

You write `.adoc` module files into an already-scaffolded showroom repo.
You MUST NOT create or modify scaffold files (`site.yml`, `ui-config.yml`, `antora.yml`, directory structure).
Scaffolding is handled by `showroom:config-helper` (RHDPCD-172) or manually before development begins.

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

After the agent finishes:

1. Verify the generated file exists in `content/modules/ROOT/pages/`
2. Check that `content/modules/ROOT/nav.adoc` includes the new module
3. Cross-check generated content sections against the module outline
4. **Update module status in `spec.yaml` to `complete`**

## Step 6: Commit

    git add content/
    git commit -m "feat: write module N — [title]"

## What You Do NOT Do

- **NEVER write AsciiDoc files directly** — always spawn rhdp-publishing-house:module-writing-helper
- **NEVER create or modify scaffold files** (`site.yml`, `ui-config.yml`, `antora.yml`, `nav.adoc` structure)
- **NEVER write modules in parallel**
- Do not review or edit content — that is the editor procedure's responsibility
