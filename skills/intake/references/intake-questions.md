# Intake Data Points — By Phase

Reference for what data points to capture in each phase. This is NOT a script — the skill
has a natural conversation and extracts these data points from it. Do not read questions
from this file verbatim.

Valid values for constrained fields come from two sources:
1. **spec.yaml inline comments** (e.g., `# cnv | aws | azure`) — read from the project
2. **Validation policy** (`~/.config/publishing-house/policy.json`) — authoritative for products, action verbs, content types

---

## Phase 1 — Discovery

Capture through conversation. Skip any field already set in spec.yaml.

| Data Point | What to Learn | spec.yaml Field |
|-----------|---------------|-----------------|
| Goal | What will someone be able to DO? Concrete, measurable outcome. | `spec.title` (derive) |
| Target audience | Role, experience level, what they know, what they don't | `spec.audience` |
| Products | Which Red Hat products and technologies. Validate against policy product list. | (used in design.md) |
| Content type | Lab (hands-on) or demo (presenter-led). Skip if pre-set. | `project.content_type` |
| Showroom type | Classic or zero-touch. Skip if pre-set. | `project.showroom_type` |
| Duration | How long end to end. | `spec.duration_hours` |
| Reference material | Existing docs, demos, blog posts. Noted, not stored in spec.yaml. | — |

---

## Phase 2 — Design Generation

Not question-based. The skill reads the design.md template from the project, fills in
sections from Phase 1 data, proposes module structure, and presents for review.

**Module structure proposal** should include:
- Module titles and estimated durations (10-30 min each)
- How modules relate (sequential, independent with shared context, fully independent)
- Total duration matching the estimate from Phase 1

---

## Phase 3 — RCARS Vetting

Not question-based. The skill queries RCARS, presents results, and discusses differentiation.

**Differentiation** captured here is stored at `approval_checklist.content.differentiation`
and pre-fills the Phase 6 approval checklist question.

---

## Phase 5 — Infrastructure Confirmation

Present a proposed profile and confirm. Only ask follow-ups for non-standard choices.

| Data Point | When to Ask | spec.yaml Field |
|-----------|-------------|-----------------|
| Cloud provider | Always (propose default from products) | `spec.environment.cloud_provider` |
| Cluster type | Always (propose based on products) | `spec.environment.cluster_type` |
| OCP version | Always (default: 4.20) | `spec.environment.ocp_version` |
| Topology | Always (default: shared-cluster) | `spec.environment.topology` |
| Worker sizing | Always (propose based on products) | `spec.environment.worker_count`, `worker_cpu`, `worker_ram_gb`, `worker_disk_gb` |
| Control plane | Auto-derived from cluster_type. Do not ask. | `spec.environment.control_plane_*` |
| Concurrent users | Only if topology = per-student or cnv-pool | `spec.environment.max_concurrent_users` |
| AI / MaaS | Only if products include AI keywords | `spec.environment.ai_requirement`, `ai_model_tier`, `ai_model_name` |
| AI justification | Only if frontier model or GPU chosen | `spec.environment.ai_justification` |
| AAP version | Only if AAP in products | `spec.environment.aap_version` |
| External services | Ask once. Accept "none." | `spec.environment.external_services` |
| Non-GA products | Only if product is beta/tech preview | `spec.environment.non_ga_products`, `non_ga_access_plan` |

**Control plane auto-derivation rules:**
- SNO: `control_plane_instance_count: 1`, `control_plane_cpu: 32`, `control_plane_ram_gb: 128`
- Multinode: `control_plane_instance_count: 3`, `control_plane_cpu: 16`, `control_plane_ram_gb: 64`

**AI keyword triggers:** AI, RHOAI, OpenShift AI, MaaS, Granite, InstructLab, Ollama, LLM, inference, model serving

---

## Phase 6 — Approval Checklist

Three fields needed for the approval checklist. Ask directly — these can't be inferred.

| Data Point | What to Ask | spec.yaml Field |
|-----------|-------------|-----------------|
| Prerequisites verifiable | What must the learner know before Module 1? Can the lab validate those automatically? | `approval_checklist.content.prerequisites_verifiable` (true/false) |
| Assessment strategy | How will we know the learner completed each module? Per-module: script, UI result, quiz, or trust-based. | `approval_checklist.content.assessment_strategy` |
| Differentiation | How does this differ from existing content? Pre-fill from Phase 3 RCARS conversation if available. | `approval_checklist.content.differentiation` |
