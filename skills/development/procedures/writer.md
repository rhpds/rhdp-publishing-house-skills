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

## Step 5: Content Commit and Verification

After the writing agent finishes:

1. Commit the written content immediately:
   ```bash
   git add content/
   git commit -m "feat: write module N — [title]"
   ```
2. Verify the generated file exists in `content/modules/ROOT/pages/`
3. Check that `content/modules/ROOT/nav.adoc` includes the new module
4. Scan for placeholders and open items:
   - Image references (`image::`) where the image file doesn't exist in `content/modules/ROOT/assets/images/`
   - Placeholder text like `TODO`, `FIXME`, `[placeholder]`, `TBD`
   - Diagram references without corresponding files
5. Collect all open items into a list

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
> **These findings are directions, not mandatory fixes.** Review them and fix what you think
> makes sense for your content. Some may not apply to your specific lab — use your judgment.
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
> **What would you like to do?**
> 1. Edit the file yourself, then say **"review again"** — I'll re-run the reviewer on your changes
> 2. Say **"module N is done"** — I'll mark it complete and we move to the next module
> 3. Ask me to fix specific items — tell me what to change and I'll update the file

**HARD STOP HERE.** Do NOT mark the module complete. Do NOT proceed to the next module.
Wait for the author to explicitly say the module is done.

## Step 5c-retry: Re-Review After Manual Edits

When the author says "review again" / "re-review" / "check it again":

1. Re-run the reviewer agent (same as Step 5b) against the current state of the `.adoc` file
2. Present updated scores and findings (same format as Step 5c)
3. HARD STOP again — wait for "module N is done" or another "review again"

This loop can repeat as many times as the author wants.

## Step 5d: Mark Complete (only after human approval)

When the author says "module N is done" / "it's done" / "mark it complete" / "looks good":

1. Re-check: verify the `.adoc` file exists and `spec.yaml` shows `status: in_progress` for this module
2. Update `publishing-house/spec.yaml`: change `status: in_progress` to `status: complete` for this module
3. Commit:
   ```bash
   git add publishing-house/spec.yaml
   git commit -m "feat: mark module N complete — [title]"
   ```
4. Confirm:
   > "Module N marked complete. [Next module available / All modules complete.]"

**Standalone completion:** If the author returns in a new session and says "module N is done" without
having run the write flow in this session, Step 5d still works — check `spec.yaml` for `status: in_progress`,
verify the `.adoc` file exists in `content/modules/ROOT/pages/`, and mark complete.

## What You Do NOT Do

- **NEVER write AsciiDoc files directly** — always spawn rhdp-publishing-house:module-writing-helper
- **NEVER create or modify scaffold files** (`site.yml`, `ui-config.yml`, `antora.yml`, `nav.adoc` structure)
- **NEVER write modules in parallel**
- **NEVER mark a module complete without human approval** — present findings and open items, then wait
- **NEVER skip the reviewer** — it runs automatically after every write

## Step 6: Generate Index and Conclusion (After All Modules Complete)

When all modules show `status: complete` in `spec.yaml`, generate the two capstone files:
**`index.adoc`** — learner-facing introduction
**`conclusion.adoc`** — recap of learning objectives and next steps

### Step 6a: Check Prerequisites

Before generating index and conclusion:

1. Verify all modules are complete: check `spec.yaml` — every module must show `status: complete`
2. Identify the target paths in `content/modules/ROOT/pages/`:
   - `00-index-learner.adoc` (or `00-index.adoc` if demo)
   - `99-conclusion.adoc`
3. Confirm `spec.yaml` module list is intact — you'll need the full list for the conclusion's "What You've Learned" recap

Present a plan before proceeding:
> "All modules are complete. I'll now generate index.adoc and conclusion.adoc to finalize your showroom."

Wait for approval — never auto-generate.

### Step 6b: Generate Index (FILE_TYPE: index)

Spawn the module-writing-helper agent:

    Task tool:
      subagent_type: rhdp-publishing-house:module-writing-helper
      prompt: |
        TARGET_FILE: content/modules/ROOT/pages/00-index-learner.adoc
        FILE_TYPE: index
        FULL_SPEC: <JSON from spec.yaml + design.md>
        LAB_TYPE: <ocp|rhel|vm|ai>
        CONTENT_TYPE: <workshop|demo>
        REPO_PATH: <absolute repo path>

Wait for the agent to complete. Verify the file exists.

Commit immediately:
```bash
git add content/modules/ROOT/pages/00-index-learner.adoc
git commit -m "feat: generate index.adoc"
```

### Step 6b-review: Auto-Run Reviewer on Index

**Do NOT skip this step.** Same pattern as module reviews (Step 5b).

Spawn the reviewer agent:

    Agent tool:
      subagent_type: rhdp-publishing-house:module-reviewer
      prompt: |
        MODULE_FILE: <absolute path to 00-index-learner.adoc>
        CONTENT_TYPE: <workshop|demo>
        LAB_TYPE: <ocp|rhel|vm|ai>
        SHARED_CONTEXT: <JSON with module_order, is_first_module: true>
        REPO_PATH: <absolute repo path>

Present findings to the author:

> **Index — Writing Complete, Awaiting Your Review**
>
> **Reviewer Score:** [score breakdown]
> **Findings:** [count] issues found
> [List HIGH and CRITICAL findings]
>
> **These findings are directions, not mandatory fixes.** Use your judgment.
>
> **Please review:** `content/modules/ROOT/pages/00-index-learner.adoc`
>
> **What would you like to do?**
> 1. Edit the file yourself, then say **"review again"**
> 2. Say **"index is done"** — I'll move on to conclusion
> 3. Ask me to fix specific items

**HARD STOP.** Wait for the author to say "index is done" or "review again".

If "review again" — re-run reviewer, present findings, HARD STOP again (same loop as Step 5c-retry).

### Step 6c: Generate Conclusion (FILE_TYPE: conclusion)

Only after the author approves index.

Spawn the module-writing-helper agent:

    Task tool:
      subagent_type: rhdp-publishing-house:module-writing-helper
      prompt: |
        TARGET_FILE: content/modules/ROOT/pages/99-conclusion.adoc
        FILE_TYPE: conclusion
        FULL_SPEC: <JSON from spec.yaml + design.md with complete module list>
        LAB_TYPE: <ocp|rhel|vm|ai>
        CONTENT_TYPE: <workshop|demo>
        REPO_PATH: <absolute repo path>

Wait for the agent to complete. Verify the file exists.

Commit immediately:
```bash
git add content/modules/ROOT/pages/99-conclusion.adoc
git commit -m "feat: generate conclusion.adoc"
```

### Step 6c-review: Auto-Run Reviewer on Conclusion

**Do NOT skip this step.** Same pattern as module reviews (Step 5b).

Spawn the reviewer agent:

    Agent tool:
      subagent_type: rhdp-publishing-house:module-reviewer
      prompt: |
        MODULE_FILE: <absolute path to 99-conclusion.adoc>
        CONTENT_TYPE: <workshop|demo>
        LAB_TYPE: <ocp|rhel|vm|ai>
        SHARED_CONTEXT: <JSON with module_order, is_conclusion: true>
        REPO_PATH: <absolute repo path>

Present findings to the author:

> **Conclusion — Writing Complete, Awaiting Your Review**
>
> **Reviewer Score:** [score breakdown]
> **Findings:** [count] issues found
> [List HIGH and CRITICAL findings]
>
> **These findings are directions, not mandatory fixes.** Use your judgment.
>
> Check that all learning objectives from your modules are captured in "What You've Learned".
>
> **Please review:** `content/modules/ROOT/pages/99-conclusion.adoc`
>
> **What would you like to do?**
> 1. Edit the file yourself, then say **"review again"**
> 2. Say **"conclusion is done"** — I'll finalize the showroom
> 3. Ask me to fix specific items

**HARD STOP.** Wait for the author to say "conclusion is done" or "review again".

If "review again" — re-run reviewer, present findings, HARD STOP again.

### Step 6d: Verify Navigation and Structure

After both index and conclusion are approved:

1. Check `content/modules/ROOT/nav.adoc` includes both files in correct order:
   - First entry: `00-index-learner.adoc` (or `00-index.adoc`)
   - Last entry: `99-conclusion.adoc`
2. Verify no placeholder text or `TODO` markers in either file
3. Check that all learning objectives from modules are consolidated in conclusion's "What You've Learned" section

### Step 6e: Mark Showroom Complete (only after human approval of both)

When both index and conclusion are approved:

1. Update `publishing-house/spec.yaml`: add a top-level field:
   ```yaml
   showroom_content_status: complete
   ```
2. Commit:
   ```bash
   git add publishing-house/spec.yaml
   git commit -m "feat: mark showroom content complete — all modules, index, and conclusion finalized"
   ```
3. Confirm:
   > "Showroom content finalized. All modules, index, and conclusion are written and reviewed."

Do NOT skip Step 6 once all modules are done. Index and conclusion are mandatory for a complete showroom.
