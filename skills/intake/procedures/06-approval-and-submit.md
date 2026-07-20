# Approval and Submit

This procedure handles the final steps: update spec.yaml with structured data,
generate supporting files, get author approval, commit, push, and submit to Central API.

## Step 1: Update spec.yaml

Update `publishing-house/spec.yaml` with structured data from the interview and spec:

```yaml
spec:
  title: "[Project Name from design.md]"
  learning_objectives:
    - "[Objective 1]"
  modules:
    - title: "Module 1 Title"
      duration_min: 20
  environment:
    ocp_version: "4.18"
    topology: "shared-cluster"
  duration_hours: 2.5
  audience: "intermediate"
```

Also update infra fields (Q12-Q18) and approval_checklist fields (Q22-Q24) gathered during intake.

## Step 2: Generate jira.yaml

Generate `publishing-house/jira.yaml` from the spec data. This file is read by Central API
after reviews complete to update the Jira epic and create child tasks. **Always overwrite
the entire file** — on re-intake (review loopback), the previous contents are replaced.

Run silently:
```bash
python3 -c "
import re
import yaml
from pathlib import Path

spec = yaml.safe_load(Path('publishing-house/spec.yaml').read_text()) or {}
project = spec.get('project', {})
spec_data = spec.get('spec', {})

title = spec_data.get('title', '') or project.get('slug', '')
content_type = project.get('content_type', 'lab')
slug = project.get('slug', '')

epic_summary = f'[PH] {title} — {content_type} ({slug})'

def extract_brief_overview(module_dir, module_index):
    pattern = f'module-{module_index:02d}-*.md'
    matches = sorted(Path(module_dir).glob(pattern))
    if not matches:
        return ''
    text = matches[0].read_text()
    m = re.search(r'## Brief Overview\s*\n(.*?)(?=\n##|\Z)', text, re.DOTALL)
    return m.group(1).strip() if m else ''

modules_dir = 'publishing-house/spec/modules'

tasks = [
    {'key': 'intake', 'summary': '[PH] Intake', 'status': 'done'},
]

modules = spec_data.get('modules', [])
for i, m in enumerate(modules, 1):
    mod_title = m.get('title', f'Module {i}')
    brief = extract_brief_overview(modules_dir, i)
    task = {
        'key': f'write-module-{i:02d}',
        'summary': f'[PH] Write Module {i}: {mod_title}',
        'status': 'open',
    }
    if brief:
        task['description'] = brief
    tasks.append(task)

tasks.append({'key': 'write-automation', 'summary': '[PH] Write Automation', 'status': 'open'})
tasks.append({'key': 'write-health-check', 'summary': '[PH] Write Health Check', 'status': 'open'})
tasks.append({'key': 'write-e2e-tests', 'summary': '[PH] Write E2E Tests', 'status': 'open'})

jira = {
    'epic': {
        'summary': epic_summary,
        'description_source': 'publishing-house/spec/design.md',
    },
    'tasks': tasks,
}

Path('publishing-house/jira.yaml').write_text(
    '# Publishing House — Jira Structure\n'
    '# Written by the intake skill, read by Central API after reviews complete.\n'
    '# Overwritten on re-intake (review loopback). Central creates tasks fresh.\n\n'
    + yaml.dump(jira, default_flow_style=False, sort_keys=False)
)
print('jira.yaml written')
"
```

## Step 3: Generate Draft Automation Manifest

Generate a draft `publishing-house/spec/automation-manifest.yaml` from the spec data.
This is a DRAFT — the automation procedure will refine it during development.

**How to derive each field:**

- **approach:** Infer from products and learning objectives:
  - If lab teaches GitOps/ArgoCD/Helm as the subject → `gitops`
  - If Ansible/AAP is the primary subject → `ansible`
  - Default for most OCP labs → `ansible`
  - If automation is mixed → `both`

- **infrastructure.type:** From topology + cloud provider:
  - `per-student` or `cnv-pool` + CNV → `ocp-cnv`
  - `per-student` + AWS → `ocp-aws`
  - `shared-cluster` + sandbox → `sandbox-tenant`
  - Default → `ocp-cnv`

- **infrastructure.ocp_version:** From `spec.environment.ocp_version`
- **infrastructure.multi_user:** `true` if max_concurrent_users > 1
- **infrastructure.users_per_deployment:** From `spec.environment.max_concurrent_users`

- **operators:** Infer from Products & Technologies section — operators the learner USES but doesn't install themselves:
  - For each: add `reason` and `source_module`
  - **Rule:** If the learner's exercise is to INSTALL the operator, do NOT list it
  - **Rule:** If the operator must exist BEFORE the learner starts, list it

- **external_services:** From `spec.environment.external_services` list

- **provision_data:** Infer from products — URLs and credentials learners need

Write the generated manifest to `publishing-house/spec/automation-manifest.yaml`.

## Step 4: Author Approval Checkpoint

Ask the author explicitly — do NOT proceed without confirmation:

> Here's what was designed for your lab. Take a moment to review `publishing-house/spec/design.md` and the module outlines in `publishing-house/spec/modules/`.
>
> **Are you happy with the design and ready to submit for review?**
> - Type **yes** (or "looks good", "proceed") to submit
> - Or give feedback and I'll update the spec

**Wait for the author's response. Do NOT auto-proceed.**

- **If feedback** → update the spec and re-validate, then ask again
- **If yes/looks good/proceed** → immediately execute Steps 5–8 WITHOUT asking again

## Step 5: Generate mkdocs.yml and TechDocs annotation

Generate `mkdocs.yml` at the repo root so RHDH TechDocs can render the spec as documentation.
Run silently:
```bash
python3 -c "
import yaml, glob, os
from pathlib import Path

spec = yaml.safe_load(Path('publishing-house/spec.yaml').read_text()) or {}
title = spec.get('spec', {}).get('title', spec.get('project', {}).get('slug', 'Publishing House Project'))

modules = sorted(glob.glob('publishing-house/spec/modules/module-*.md'))
index_lines = [f'# {title}', '', 'Welcome to the project spec. Use the navigation to browse the design and module outlines.', '', '- [Design Spec](design.md)']
for m in modules:
    fname = os.path.basename(m)
    parts = fname.replace('.md', '').split('-', 2)
    num = int(parts[1]) if len(parts) > 1 else 0
    label = parts[2].replace('-', ' ').title() if len(parts) > 2 else fname
    index_lines.append(f'- [Module {num} - {label}](modules/{fname})')
Path('publishing-house/spec/index.md').write_text(chr(10).join(index_lines) + chr(10))

nav = [{'Home': 'index.md'}, {'Design Spec': 'design.md'}]
if modules:
    mod_nav = []
    for m in modules:
        fname = os.path.basename(m)
        parts = fname.replace('.md', '').split('-', 2)
        num = int(parts[1]) if len(parts) > 1 else 0
        label = parts[2].replace('-', ' ').title() if len(parts) > 2 else fname
        mod_nav.append({f'Module {num} - {label}': f'modules/{fname}'})
    nav.append({'Modules': mod_nav})

if Path('publishing-house/spec/automation-manifest.yaml').exists():
    nav.append({'Automation Manifest': 'automation-manifest.yaml'})

mkdocs = {
    'site_name': title,
    'docs_dir': 'publishing-house/spec',
    'nav': nav,
    'plugins': ['techdocs-core'],
}
with open('mkdocs.yml', 'w') as f:
    yaml.dump(mkdocs, f, default_flow_style=False, sort_keys=False)
print('mkdocs.yml created')
"
```

The `backstage.io/techdocs-ref` annotation is already in `catalog-info.yaml` from the project template.

## Step 6: Commit and push

```bash
git add publishing-house/ mkdocs.yml catalog-info.yaml
git commit -m "feat: intake complete — design spec, module outlines, and jira structure"
git push
```

**Run this immediately. Do NOT ask the author.**

## Step 7: Submit intake to Central API

```bash
python publishing-house/tools/ph-intake.py 2>&1
```

**Run this immediately. Do NOT ask the author.**

The response is always the same shape:
```json
{"status": <int>, "stage": "<stage or null>", "error": "<msg or null>", "validation": <object or null>}
```

Parse by `status`:

- **201** — Success. Show: "Intake submitted. Stage is now **{stage}**."
- **422** — Validation failed. `validation.results` contains the failed checks.
  - Show each failed check to the author: check ID, message, and field path
  - Help the author fix the issues
  - After fixes, commit, push, and re-run `ph-intake.py`
  - Loop until validation passes
- **409** — Workflow is not in intake stage. Show `error` and **STOP.**
- **404** — No workflow found. Show `error` and **STOP.**
- **Any other status** — Show the `error` message and **STOP.**

## Step 7b: Project structure cleanup

Check `project.showroom_type` in spec.yaml:

- **If `classic`** (or empty/unset): Remove Zero-Touch directories silently:
  ```bash
  rm -rf runtime-automation/ setup-automation/
  git add -A runtime-automation/ setup-automation/ 2>/dev/null || true
  git commit -m "chore: remove zero-touch dirs (classic showroom)" 2>/dev/null || true
  git push 2>/dev/null || true
  ```
- **If `zero_touch`**: Keep `runtime-automation/` and `setup-automation/` in place.

## Step 8: Report result

Parse `stage` from the JSON response.

- If the call succeeds → show: "Intake submitted. Stage is now **{stage}**."
- If the call fails → show the error from the response.

**Return to the orchestrator** (if dispatched) or **STOP** (if invoked directly).
