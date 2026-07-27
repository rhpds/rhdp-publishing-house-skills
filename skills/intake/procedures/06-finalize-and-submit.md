# Finalize and Submit

Phase 6 of the intake flow. Complete the approval checklist, generate supporting files,
validate, and submit to Central API.

## Step 1: Approval Checklist

Ask the remaining approval checklist questions. These are the fields reviewers need.

**Prerequisites verifiable:**
> "What must the learner know or have done before starting Module 1? And can the lab
> automatically validate those prerequisites when the learner starts?"

Write to `approval_checklist.content.prerequisites_verifiable` (true/false).

**Assessment strategy (optional — ask only for Zero-Touch labs):**

If `project.showroom_type` is `zero_touch`, ask:
> "Since this is a Zero-Touch lab, how should each module be validated? Per module:
> solve/validate button, verification script, or automated check."

Write to `approval_checklist.content.assessment_strategy`.

If classic showroom or demo → skip this question.

**Differentiation:**
If `approval_checklist.content.differentiation` is already populated from Phase 3
(RCARS vetting), present it for confirmation:
> "Based on our RCARS discussion, here's the differentiation statement: *'{differentiation}'*
> Does this capture it, or would you adjust the wording?"

If empty (RCARS was skipped or offline) → leave it empty. Do NOT ask the author to
explain differentiation without RCARS context. The field will be populated when RCARS
vetting runs (either in a later session or when Central validates at submission).

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

Derive each field from what's already in spec.yaml and design.md:
- **approach:** ansible (default for most OCP labs), gitops (if lab teaches GitOps), both
- **infrastructure.type:** From topology + cloud provider (e.g., `per-student` + CNV → `ocp-cnv`)
- **operators:** Infer from Products & Technologies — operators the learner USES but doesn't install
- **external_services:** From `spec.environment.external_services`

## Step 4: Generate mkdocs.yml

Generate `mkdocs.yml` at the repo root for RHDH TechDocs rendering.

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

## Step 5: Author Checkpoint

> "Your spec is ready for submission. Take a moment to review
> `publishing-house/spec/design.md` and the module outlines in
> `publishing-house/spec/modules/`.
>
> **Are you happy with this and ready to submit for review?**"

**Wait for the author's response. Do NOT auto-proceed.**

- **If feedback** → update the spec, re-validate, ask again
- **If approved** → immediately execute Steps 6-8 WITHOUT asking again

## Step 6: Commit and push

```bash
git add publishing-house/ mkdocs.yml catalog-info.yaml
git commit -m "feat: intake complete — design spec, module outlines, and jira structure"
git push
```

**Run this immediately. Do NOT ask the author.**

## Step 7: Submit intake to Central API

**If offline → defer:**
> "Your spec is complete and committed. When the platform is available, run
> `/rhdp-publishing-house` again to submit through the review gate."
>
> **STOP.**

**If online:**

```bash
python publishing-house/tools/ph-intake.py 2>&1
```

**Run this immediately. Do NOT ask the author.**

The response shape:
```json
{"status": <int>, "stage": "<stage or null>", "error": "<msg or null>", "validation": <object or null>}
```

Parse by `status`:

- **201** — Success. Show: "Intake submitted successfully. Run `/rhdp-publishing-house` to check the current stage."
- **422** — Validation failed. `validation.results` contains the failed checks.
  - Show each failed check to the author: check ID, message, and field path
  - Propose specific fixes for each issue and **ask the author to confirm before making any changes**
  - Do NOT edit files, commit, or push until the author approves the proposed fixes
  - After the author approves and fixes are applied, commit, push, and re-run `ph-intake.py`
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

- If submission succeeded → show: "Intake submitted successfully. Run `/rhdp-publishing-house` to check the current stage."
- If submission failed → show the error from the response.

**Return to the orchestrator** (if dispatched) or **STOP** (if invoked directly).
