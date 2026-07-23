# Infrastructure Confirmation

Phase 5 of the intake flow. Capture infrastructure requirements as a single confirm-or-adjust interaction.

## Derive Defaults

Read `publishing-house/spec.yaml` inline comments to understand valid values for each field.
Derive sensible defaults from what you already know:

| Signal from Discovery | Default |
|----------------------|---------|
| Products include OCP Virtualization/CNV | `cloud_provider: cnv`, `cluster_type: multinode` |
| Products include AAP | Note: `aap_version` will be needed |
| Products include AI keywords (RHOAI, MaaS, Granite, InstructLab, Ollama, LLM, inference, model serving) | `ai_requirement: maas`, `ai_model_tier: open-source` |
| No AI keywords | `ai_requirement: none` |
| No specific cloud reason | `cloud_provider: cnv` (platform default) |
| No topology discussed | `topology: shared-cluster` |
| OCP version not specified | `ocp_version: "4.20"` (minimum) |

## Present as One Profile

Present a complete infrastructure profile for confirmation — not individual questions:

> "Based on your products, here's what I'd suggest for infrastructure:
>
> - **Cloud provider:** CNV
> - **Cluster type:** Multinode, 6 workers (8 vCPU, 32GB RAM, 100GB disk)
> - **OCP version:** 4.20
> - **Topology:** Shared cluster
> - **AI/MaaS:** None
> - **External services:** None
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
- **Concurrent users:** Only if topology is per-student or cnv-pool.
- **External services:** Ask once. Accept "none."

## Write Point

Write all infrastructure fields to spec.yaml and update the Infrastructure Requirements
section of design.md:

```bash
git add publishing-house/spec.yaml publishing-house/spec/design.md
git diff --cached --quiet || git commit -m "feat: phase 5 — infrastructure confirmed" 2>/dev/null || true
```

Proceed to Phase 6 (finalize + submit).
