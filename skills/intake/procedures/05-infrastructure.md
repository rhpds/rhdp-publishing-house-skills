# Infrastructure Confirmation

Phase 5 of the intake flow. Capture infrastructure requirements as a single confirm-or-adjust interaction.

## Derive Defaults

Read `publishing-house/spec.yaml` inline comments to understand valid values for each field.
Derive sensible defaults from what you already know:

| Signal from Discovery | Default |
|----------------------|---------|
| Products include OCP / OpenShift | `cloud_provider: cnv`, `cluster_type: multinode`, propose `ocp_version` |
| Products include OCP Virtualization/CNV | `cloud_provider: cnv`, `cluster_type: multinode` |
| Products include AAP (without OCP) | `cloud_provider: cnv` (VMs on RHDP infra), no `ocp_version` — AAP runs on RHEL |
| Products include AI keywords (RHOAI, MaaS, Granite, InstructLab, Ollama, LLM, inference, model serving) | `ai_requirement: maas`, `ai_model_tier: open-source` |
| No AI keywords | `ai_requirement: none` |
| No specific cloud reason | `cloud_provider: cnv` (platform default) |
| No topology discussed | `topology: shared-cluster` |

**OCP version:** Only propose an OCP version when OpenShift is one of the products the
learner interacts with in the lab. If the lab is AAP-on-RHEL, VM-based, or otherwise
doesn't involve OpenShift, leave `ocp_version` blank — the infra reviewer will set the
platform version if needed. Do NOT make up reasons why OCP version is relevant when the
lab doesn't use OpenShift.

## Present as One Profile

Present a complete infrastructure profile for confirmation — not individual questions.
Only include fields that are relevant to this lab's products:

> "Based on your products, here's what I'd suggest for infrastructure:
>
> - **Cloud provider:** CNV
> - **Topology:** Per-student
> - [include **OCP version** ONLY if OpenShift is a product in this lab]
> - [include other relevant fields]
>
> Does this look right, or should I adjust anything?
> You can also edit `spec.yaml` directly if you prefer."

## Conditional Follow-ups

Only ask these if triggered — do not ask them by default:

- **AI/MaaS:** Only if products include AI keywords. Default to MaaS + open-source.
  If author chooses frontier model or GPU: "Why is an open-source model insufficient?"
- **AAP version:** Only if AAP is in the products list.
- **Non-GA products:** Only if any product is labeled beta, tech preview, or early access.
  If non-empty: "How will access be provided during provisioning?"
- **Concurrent users:** Only if topology is shared-cluster. Shared clusters need to be
  sized for the number of simultaneous users. Per-student and cnv-pool topologies give
  each learner their own environment — the number of simultaneous users is an operational
  decision made at scheduling time, not during intake. Do NOT ask the author how many
  people will be in a room or at an event.
- **External services:** Ask once. Accept "none."

## Confirmation Required

**After the author provides adjustments, present the updated profile and wait for
explicit confirmation before proceeding.** Do NOT silently apply changes and move on.

> "Here's the updated infrastructure profile: [show updated fields].
> Does this look good to move forward?"

Only proceed to Phase 6 after the author confirms.

## Rules

- Do NOT fabricate explanations for why a field is relevant when it isn't.
- If the author challenges a field, remove it or leave it blank. Do not invent justifications.
- Only propose fields that are directly relevant to the products and design.
- If you're unsure whether a field applies, leave it for the infra reviewer.

## Write Point

Write all infrastructure fields to spec.yaml and update the Infrastructure Requirements
section of design.md:

```bash
git add publishing-house/spec.yaml publishing-house/spec/design.md
git diff --cached --quiet || git commit -m "feat: phase 5 — infrastructure confirmed" 2>/dev/null || true
```

Proceed to Phase 6 (finalize + submit).
