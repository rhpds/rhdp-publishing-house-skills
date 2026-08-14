---
name: rhdp-publishing-house:automation-helper
description: This skill should be used when the user asks to "build automation", "write the Ansible roles", "set up GitOps", "capture automation requirements", or "generate catalog configuration". Optional helper that handles lifecycle automation — requirements capture, catalog entry configuration, and environment automation code. Not mandatory — authors may build automation themselves instead.
context: main
---

# Automation Helper

You handle lifecycle automation phases: capturing automation requirements (7a), generating
catalog entry configuration (7b), and developing environment automation code (7c).
You dispatch to `rhdp-publishing-house:ansible-helper` and `rhdp-publishing-house:gitops-helper`
based on the automation approach in the manifest.

This is an **optional helper**. Authors are never required to use it — they may build and manage
their own automation directly. It has no bearing on module status tracking or submission to
Central, which are the `rhdp-publishing-house:development` skill's job.

See @rhdp-publishing-house/skills/automation-helper/references/automation-patterns.md for automation patterns.
See @rhdp-publishing-house/skills/automation-helper/references/ansible-automation-guide.md for Ansible collection structure.
See @rhdp-publishing-house/skills/automation-helper/references/gitops-automation-guide.md for GitOps (Helm + ArgoCD) patterns.
See @rhdp-publishing-house/skills/automation-helper/references/automation-manifest-format.md for the automation manifest format.
See @rhdp-publishing-house/skills/development/references/gitops-patterns.md for GitOps conventions and reference repo patterns.

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

See @rhdp-publishing-house/skills/automation-helper/references/automation-manifest-format.md for the full manifest format.

## Phase 7b: Catalog Configuration

**self_published:** Skip this phase automatically — no catalog entry needed.

**rhdp_published:** Dispatch to `rhdp-publishing-house:ansible-helper` (FUTURE — RHDPCD-110)
to generate catalog entry configuration from the approved automation manifest.
Do NOT call agnosticv skills — catalog automation is handled by PH-native skills only.

## Phase 7c: Automation Code

Dispatch to the appropriate PH skill based on `automation_approach` in the manifest:

- `ansible`  → Skill tool: `rhdp-publishing-house:ansible-helper`  (FUTURE — RHDPCD-110)
- `gitops`   → Skill tool: `rhdp-publishing-house:gitops-helper`
- `both`     → ansible-helper first, then gitops-helper

The automation directories (`automation/bootstrap-infra/`, etc.) are created by the config-helper's
Automation Scaffolding during initial project scaffolding. The gitops-helper populates them with
real workloads.

**FUTURE:** The Ansible helper skill does not exist yet. Until it is created, refer to the reference
guides below and write Ansible automation collaboratively with the author using the manifest as the spec.

See the detailed guides for code structure and patterns.

After writing, run safety check and create worklog entries for any blockers.

## Phase 7d: Testing (Gate)

Present testing instructions. Wait for user to describe what was tested. Record result.
This is a human gate — the agent does not deploy or test automation itself. The author manages
their own testing; nothing here blocks module completion or Central submission.

## What You Do NOT Do

- Do not write Showroom content — that is `rhdp-publishing-house:writer-helper`'s job
- Do not review content quality — that is `rhdp-publishing-house:reviewer-helper`'s job
- Do not deploy or test the automation yourself
- Do not track module status, mark the showroom complete, or submit to Central — that is the
  `rhdp-publishing-house:development` skill's job
- Do not advance the lifecycle phase — only update substep status
