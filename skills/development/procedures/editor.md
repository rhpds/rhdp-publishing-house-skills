# Editor

You perform technical editing by spawning `rhdp-publishing-house:module-reviewer` and adding
Publishing House-specific spec alignment checks. You verify content quality AND
alignment with the approved project spec.

**Note:** The reviewer agent is automatically invoked by `writer.md` after every module write (Step 5b).
This editor procedure is for **standalone re-reviews** — when the author wants to re-check content
after making manual edits, or when reviewing content that was written outside the PH workflow.

See @rhdp-publishing-house/skills/development/references/editing-checklist.md for the full editing checklist.

## Step 1: Determine Scope

Check what the user requested:

- **"edit module N"** → review that specific module
- **"edit all" / "review content" / "technical edit"** → review all drafted modules
- **"edit"** with no qualifier → review the next un-reviewed drafted module

**If no modules are drafted:**
> "No drafted modules found. The writing phase needs to produce content before editing can begin. Would you like to write a module first?"

## Step 2: Spawn rhdp-publishing-house:module-reviewer Agent

For each module to review, spawn the agent via Task tool:

    Task tool:
      subagent_type: rhdp-publishing-house:module-reviewer
      prompt: |
        MODULE_FILE: content/modules/ROOT/pages/<module-filename>.adoc
        CONTENT_TYPE: <workshop|demo>
        LAB_TYPE: <ocp|rhel|vm|ai>
        SHARED_CONTEXT: <JSON with module_order, defined_attributes, cross-module refs>
        REPO_PATH: <absolute repo path>

## Step 3: Run Spec Alignment Checks

After `rhdp-publishing-house:module-reviewer` completes, run PH-specific checks.

Read the module outline from `publishing-house/spec/modules/module-NN-*.md`.
Read the generated content from the content directory.
Read `publishing-house/spec/design.md` for project-level context.

### SA-1: Outline Coverage
### SA-2: Learning Objectives Match
### SA-3: Duration Alignment
### SA-4: Cross-Module Consistency (only when reviewing multiple modules)
### RS-1: Product Name Accuracy
### RS-2: Version Consistency

See @rhdp-publishing-house/skills/development/references/editing-checklist.md for details.

## Step 4: Produce Review Report

Write the review report to `publishing-house/reviews/editing-review-module-NN.md`.

## Step 5: Fix Loop

Enter fix loop based on autonomy:

- **Supervised:** Present report, ask which issue to fix first.
- **Semi:** Auto-fix MEDIUM/LOW, present CRITICAL/HIGH for decision.
- **Full:** Auto-fix all with clear fixes, present judgment calls.

After fixes, re-run spec alignment checks to verify.

## What You Do NOT Do

- Do not write new content — that is the writer procedure's responsibility
- Do not modify the module outlines without user confirmation
- Do not advance the lifecycle phase
- Do not create or modify scaffold files — scaffolding is Andrew's domain (RHDPCD-172)
