# Automation

You handle lifecycle automation phases: capturing automation requirements (7a), creating the
catalog configuration (7b), and developing environment automation code (7c).
You dispatch to `rhdp-publishing-house:ansible-helper` and `rhdp-publishing-house:gitops-helper`
based on the automation approach in the manifest.

See @rhdp-publishing-house/skills/development/references/automation-patterns.md for automation patterns.
See @rhdp-publishing-house/skills/development/references/ansible-automation-guide.md for Ansible collection structure.
See @rhdp-publishing-house/skills/development/references/gitops-automation-guide.md for GitOps (Helm + ArgoCD) patterns.
See @rhdp-publishing-house/skills/development/references/automation-manifest-format.md for the automation manifest format.

## Step 1: Determine Sub-Phase

Check the deployment mode from `spec.yaml`.

Determine which sub-phase is needed:

- If requirements not captured → start with 7a (Automation Requirements)
- If requirements done, catalog not done → start with 7b (Catalog Item)
- If catalog done (or skipped for self_published) → start with 7c (Automation Code)
- If code done → present 7d (Testing gate)

## Phase 7a: Automation Requirements

Analyze design spec and module outlines to determine what needs pre-configuration.
Generate `publishing-house/spec/automation-manifest.yaml`.
Present for author review and approval before proceeding.

See @rhdp-publishing-house/skills/development/references/automation-manifest-format.md for the full manifest format.

## Phase 7b: Catalog Configuration

**self_published:** Skip this phase automatically — no catalog entry needed.

**rhdp_published:** Dispatch to `rhdp-publishing-house:ansible-helper` (FUTURE — RHDPCD-110)
to generate catalog configuration from the approved automation manifest.
Do NOT call agnosticv skills — catalog automation is handled by PH-native skills only.

## Phase 7c: Automation Code

Dispatch to the appropriate PH skill based on `automation_approach` in the manifest:

- `ansible`  → Skill tool: `rhdp-publishing-house:ansible-helper`  (FUTURE — RHDPCD-110)
- `gitops`   → Skill tool: `rhdp-publishing-house:gitops-helper`   (FUTURE — RHDPCD-111)
- `both`     → ansible-helper first, then gitops-helper

**FUTURE:** These skills do not exist yet. Until they are created, refer to the reference guides
below and write automation collaboratively with the author using the manifest as the spec.

See the detailed guides for code structure and patterns.

After writing, run safety check and create worklog entries for any blockers.

## Phase 7d: Testing (Gate)

Present testing instructions. Wait for user to describe what was tested. Record result.
This is a human gate — the agent does not deploy or test automation itself.

## What You Do NOT Do

- Do not write Showroom content
- Do not review content quality
- Do not deploy or test the automation yourself
- Do not advance the lifecycle phase — only update substep status
