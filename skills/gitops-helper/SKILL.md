---
name: rhdp-publishing-house:gitops-helper
description: This skill should be used when the user asks to "write GitOps automation", "create Helm charts", "set up ArgoCD for this lab", "generate GitOps manifests", "build GitOps deployment", or "deploy with Helm and ArgoCD". Populates existing GitOps automation directories (Helm + ArgoCD) with real workloads for RHDP lab environments following rhdp-gitops-patterns conventions.
context: main
---

# GitOps Helper

You populate existing GitOps automation directories with real workloads (Helm templates + ArgoCD
manifests) for RHDP lab and demo environments. You follow the conventions in
@rhdp-publishing-house/skills/gitops-helper/references/gitops-patterns.md.

The `automation/gitops/bootstrap-infra/` (and optionally `bootstrap-tenant/`) directories are
created by the config-helper's Automation Scaffolding during initial project scaffolding. This
skill works with those existing directories — it does NOT create them.

## Tool Boundaries

You work locally: read files, write files, run `helm template` for validation.
**Do NOT use** MCP tools or call external APIs directly.
In Publishing House mode, all backend interactions go through `publishing-house/tools/` scripts.
**If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately** —
show the error output and do not continue.

## Mode Detection

Detect the mode before doing anything else:

```bash
test -f catalog-info.yaml && test -f publishing-house/spec.yaml && echo "ph" || echo "standalone"
```

- `ph` → **Publishing House mode**. Start at Step 1.
- `standalone` → **Standalone mode**. Skip to Step 3.

Both modes run Steps 3–8. The only difference is that standalone skips pre-flight (1)
and workflow check (2).

---

## Steps 1–2 — Publishing House mode only

### Step 1 — Pre-flight

Follow @rhdp-publishing-house/skills/common/pre-flight.md (Steps 1–3: verify project, read identity, check auth).

### Step 2 — Workflow check

**RULE: This sequence runs every invocation. No exceptions. No skipping.**

**2a.** Get workflow data:
```bash
python publishing-house/tools/ph-workflow-data.py
```
If this fails → set `offline_mode = true`, skip to Step 3.
If this succeeds → extract `workflow_id`. Set `offline_mode = false`.

**2b.** Get workflow state (skip if offline):
```bash
python publishing-house/tools/ph-workflow-state.py WORKFLOW_ID
```
If stage is not `development` → STOP. Tell the author this skill runs during the development stage.
If offline → assume `development`.

**2c.** Sync (skip if offline):
```bash
python publishing-house/tools/ph-sync.py
```

---

## Step 3 — Verify automation directories exist

Check that the automation directories are already scaffolded:

```bash
test -d automation/gitops/bootstrap-infra && echo "infra:yes" || echo "infra:no"
test -d automation/gitops/bootstrap-tenant && echo "tenant:yes" || echo "tenant:no"
```

- `infra:no` → STOP. Tell the author:
  > "The `automation/gitops/bootstrap-infra/` directory doesn't exist yet. Run the
  > **development** skill and select **GitOps Automation** to scaffold the automation
  > directories first, then come back here to populate them with workloads."
- `infra:yes` → proceed. Note whether `tenant:yes` for later steps.

## Step 4 — Clone reference repo

Clone the RHDP GitOps patterns repo. This is required — the examples
are the foundation for all generated automation.

```bash
git clone --depth 1 https://github.com/rhpds/rhdp-gitops-patterns.git /tmp/rhdp-gitops-patterns 2>&1
```

**If the clone fails → STOP.** Tell the user:
> "Cannot clone the rhdp-gitops-patterns reference repo. This skill requires it for
> examples. Check your network connection and try again."

## Step 5 — Additional reference repos

Use `AskUserQuestion` with two options:

- **"Proceed with rhdp-gitops-patterns (Recommended)"** — continue with the default repo only.
- **"Add additional reference repos"** — free text where the user can provide extra repo URLs
  to use as examples alongside the default patterns repo.

Additional repos are for **examples only** — they do not replace the default
examples. Clone each one to `/tmp/user-gitops-ref-N/`:

```bash
git clone --depth 1 REPO_URL /tmp/user-gitops-ref-1 2>&1
```

If any additional clone fails, warn the user and continue with whatever succeeded.

## Step 6 — Gather inputs

Collect inputs in this priority order, combining all sources:

### 6a. Skill arguments

Check if the user passed arguments when invoking the skill.

### 6b. Project context (PH mode only)

If in a Publishing House project:
- Read `publishing-house/spec.yaml` for project metadata (products, platform, ocp_version, topology, automation_type)
- Read `publishing-house/spec/design.md` for the full design spec
- Read module outlines in `publishing-house/spec/modules/` for what needs pre-configuration

### 6c. Clarifying questions

After analyzing all available inputs, determine what is still unclear or missing.
Ask the user clarifying questions for anything you cannot determine from the inputs:
- What namespaces does each user need?
- What applications, operators, or services should be pre-configured?
- Are there any VMs (KubeVirt)?
- Does the user need their own ArgoCD instance?

Always combine all inputs — arguments, project context, and user answers.

## Step 7 — Populate templates

### 7a. Classify resources

For each component from the inputs, decide:
- **Infra** if cluster-wide or shared (operators, shared services) → `automation/gitops/bootstrap-infra/templates/`
- **Tenant** if per-user (applications, VMs, RBAC, seed data) → `automation/gitops/bootstrap-tenant/templates/`

If a resource looks tenant-scoped but `automation/gitops/bootstrap-tenant/` does not exist, warn the user:
> "This resource looks per-user but there's no tenant chart. Should I create
> `automation/gitops/bootstrap-tenant/`, or place this in infra?"

If the user wants tenant, copy the tenant scaffold from `.scaffolds/automation/gitops/bootstrap-tenant/`
into `automation/gitops/bootstrap-tenant/`.

### 7b. Generate templates

For each component:

1. **Check reference repo examples first.** Search the default `rhdp-gitops-patterns/examples/`
   and any additional user-provided repos. If a matching pattern exists (e.g., GitLab, DevHub,
   Istio Gateway, per-user ArgoCD, KubeVirt VMs), use it as the basis for generation.
   Adapt values and namespaces to the current project. Add a provenance comment to each
   generated file with the full git repo URL of the source, e.g.
   `# Generated from https://github.com/redhat-gpte/rhdp-gitops-patterns/examples/modernize-ocp-virt`.

   **Never generate ArgoCD Application CRs that point back to subdirectories of the same repo.**
   Expand all manifests directly into the chart templates.

2. **If no example matches**, ask the user: "I don't have a reference pattern for deploying
   [component]. Can you point me to a repo or manifests I can use as a starting point?"

3. **If the user has no reference**, generate templates based on best practices and the
   information available. Tell the user what you assumed and ask for review.

Apply sync-wave annotations per the ordering conventions in `gitops-patterns.md`.

**Before generating any operator Subscription**, check the "Known Operator Quirks"
section in `gitops-patterns.md`. Some operators (e.g., Gitea) are not in standard
OLM catalogs and require a custom CatalogSource or specific install modes. Always
apply these requirements during generation -- do not rely on deployment-time debugging.

**Before using any S2I builder image** (ubi9/nginx-122, ubi9/httpd-24, ubi9/python-311),
check the "S2I Builder Images" section in `gitops-patterns.md`. These images require
a command override and content mount -- they will CrashLoopBackOff if deployed bare.

**For PVCs**, place them at the same sync-wave as the workload that uses them
(not with namespaces at wave -2). See "PVC with WaitForFirstConsumer StorageClass"
in `gitops-patterns.md`.

Ensure all tenant resources target one of the tenant's namespaces (never a shared namespace).

### 7c. Present for review

Show the user what was generated:
- List of files created
- Summary of what goes in infra vs tenant
- Any assumptions made
- Any components that need user-provided references

Wait for the author to review the generated files.

## Step 8 — Review and next steps

Tell the user to review, stage, and commit the generated files themselves.

Then print a suggested AgnosticV `common.yml` snippet to the console.

**Always print this warning first:**
> **AgnosticV config suggestion** — review and test before using. This is only a starting point.

Generate a snippet for the cluster catalog item (`bootstrap-infra`):
```yaml
ocp4_workload_gitops_bootstrap_repo_url: https://github.com/ORG/REPO
ocp4_workload_gitops_bootstrap_repo_revision: "{{ gitops_repo_revision }}"
ocp4_workload_gitops_bootstrap_repo_path: automation/gitops/bootstrap-infra
ocp4_workload_gitops_bootstrap_application_name: bootstrap-infra
ocp4_workload_gitops_bootstrap_helm_values:
  # Only include values prone to external changes.
  # deployer.domain, deployer.apiUrl, and deployer.guid are auto-injected.
  ...
```

If `automation/gitops/bootstrap-tenant` exists, also print a tenant snippet:
```yaml
ocp4_workload_gitops_bootstrap_repo_url: https://github.com/ORG/REPO
ocp4_workload_gitops_bootstrap_repo_revision: "{{ gitops_repo_revision }}"
ocp4_workload_gitops_bootstrap_repo_path: automation/gitops/bootstrap-tenant
ocp4_workload_gitops_bootstrap_application_project: tenants
ocp4_workload_gitops_bootstrap_application_name: "bootstrap-{{ guid }}"
ocp4_workload_gitops_bootstrap_helm_values:
  username: "{{ ocp4_workload_user_base }}{{ user_num }}"
  ...
```

Populate the `helm_values` block with only the values that should be deployer-managed:
operator channels/CSVs, git revisions, image tags, secrets, user count/prefix.
Leave everything else to the chart's `values.yaml` defaults.

## Step 9 — Completion confirmation (Publishing House mode only)

Skip this step in standalone mode.

After the user has reviewed the generated files (Step 8), confirm:
> "GitOps automation is generated. You can come back to add more workloads anytime.
> Returning to development dashboard."

Do not mark automation complete or close Jira tickets — the development skill handles that.

## Rules

- The rhdp-gitops-patterns repo is required. If it cannot be cloned, STOP.
- The automation directories must already exist (created by config-helper scaffolding). This skill does not create them.
- The automation manifest is an input, not a contract. Accept whatever format and fields are there.
- Never place tenant resources in shared namespaces.
- Check examples before generating from scratch.
- Ask the user when you don't have a reference for a component.
- Do not hardcode cluster domains — construct URLs from `deployer.domain`.
- Never enable the ApplicationSet in `bootstrap-infra`. Do not add a `tenant:` key to its `values.yaml`. The ApplicationSet is for manual use only.
- Do not advance the lifecycle phase — that is the development skill's job. This skill only marks the automation workstream complete (Step 9).
