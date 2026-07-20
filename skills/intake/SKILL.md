---
name: rhdp-publishing-house:intake
description: This skill should be used when the user asks to "create a spec", "write a design doc", "start a new lab project", "I have an idea for a lab", "I have a Jira issue with requirements", or "pull requirements from Jira". It handles intake for RHDP Publishing House projects.
---

---
context: main
model: claude-opus-4-6
---

# Intake Agent

**RULE: If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately.** Show the error output to the author and say there was an issue calling the backend. Do not continue the skill.

You handle the intake phase of the Publishing House lifecycle. This skill is
self-sufficient — it works whether dispatched by the orchestrator or invoked directly.

## Tool Boundaries

**Do NOT use** Central API tools directly. You work locally: read files, write specs, update spec.yaml.

**Do NOT use** MCP tools. All external interactions go through `publishing-house/tools/` scripts.

## Pre-flight

**ALWAYS complete these steps first.**

### Step 1 — Verify this is a Publishing House project

Run silently:
```bash
python3 -c "
from pathlib import Path
ci = Path('catalog-info.yaml')
spec = Path('publishing-house/spec.yaml')
if ci.exists() and spec.exists():
    print('ok')
elif not ci.exists():
    print('no-catalog')
else:
    print('no-spec')
"
```

- `ok` → proceed.
- `no-catalog` → show:
  > This doesn't look like a Publishing House project — `catalog-info.yaml` is missing.
  >
  > Projects must be created through the **RHDH Developer Hub** template. Open RHDH, choose the **Publishing House Content Project** template, and fill in the form.

  **STOP.**
- `no-spec` → show: "`publishing-house/spec.yaml` is missing. This repo may not have been scaffolded correctly." **STOP.**

### Step 2 — Read project identity

Run silently:
```bash
python3 -c "
import yaml
from pathlib import Path
spec = yaml.safe_load(Path('publishing-house/spec.yaml').read_text()) or {}
pid = spec.get('project', {}).get('slug', '')
print(f'project_id:{pid}')
"
```

Extract `project_id`. If empty → show error: "`project.slug` is missing in `spec.yaml`." **STOP.**

### Step 3 — Check auth

Run silently:
```bash
python3 -c "
import json, os
f = os.path.expanduser('~/.config/publishing-house/auth.json')
if os.path.exists(f):
    d = json.load(open(f))
    cred = d.get('credential', '')
    central = d.get('central', '')
    print(f'cred:{cred[:8]}' if cred else 'no-cred')
    print(f'central:{central}')
else:
    print('missing')
"
```

Extract `central_url` from the `central:` line.

- Has `cred:` and `central:` → proceed.
- `missing` → show: "Workspace auth is not configured. Restart the DevSpaces workspace to trigger setup." **STOP.**
- `no-cred` → show:

  > **You need a Publishing House API key.**
  >
  > Open this URL in your browser:
  > **`{central_url}`**
  >
  > Log in with your Red Hat SSO, click **Generate New Key**, and **paste the key here** — I'll save it for you.

  Then try to open the browser:
  ```bash
  python3 -c "import subprocess; subprocess.Popen(['open', 'CENTRAL_URL'])" 2>/dev/null || true
  ```
  Replace CENTRAL_URL with the actual `central_url`.

  Wait for the author to paste the key. Once received, save it:
  ```bash
  python3 -c "
import json, os
key = 'PASTE_KEY_HERE'
path = os.path.expanduser('~/.config/publishing-house/auth.json')
d = json.load(open(path)) if os.path.exists(path) else {}
d['credential'] = key
os.makedirs(os.path.dirname(path), exist_ok=True)
with open(path, 'w') as f:
    json.dump(d, f, indent=2)
os.chmod(path, 0o600)
print('saved')
"
  ```
  Replace PASTE_KEY_HERE with the actual key. Confirm: > Got it — you're all set.

### Step 4 — Get workflow state

First check spec.yaml for a cached workflow_id:
```bash
python3 -c "
import yaml
from pathlib import Path
spec = yaml.safe_load(Path('publishing-house/spec.yaml').read_text()) or {}
wfid = spec.get('project', {}).get('workflow_id', '')
print(f'workflow_id:{wfid}')
"
```

**If workflow_id is present** → just get the stage:
```bash
python publishing-house/tools/ph-workflow-state.py WORKFLOW_ID
```
Replace WORKFLOW_ID with the extracted value. Extract `stage`. No file writes needed.

**If workflow_id is blank** → run sync to fetch and persist it:
```bash
python publishing-house/tools/ph-sync.py
```
Extract `stage`, `workflow_id`, `epic_key`, and `synced` from the output.
If `synced:true`, commit silently:
```bash
git add publishing-house/spec.yaml catalog-info.yaml
git diff --cached --quiet || git commit -m "feat: sync workflow data from Central API" 2>/dev/null || true
```

If stage is not `intake` → show:
> Cannot start this skill because the project is in **{stage}** stage. This skill requires **intake**.

**STOP — do not proceed.**

### Step 5 — Load policy and references

1. Fetch validation policy:
   ```bash
   python publishing-house/tools/ph-policy.py
   ```
   If it fails, show the error and **STOP**.

2. Read `~/.config/publishing-house/policy.json`. Use these lists throughout intake:
   - `valid_content_types` — accept only these when the author states a content type
   - `valid_audiences` — accept only these for difficulty/audience
   - `products` (with `aliases`) — validate product names against this list
   - `action_verbs_valid` — learning objectives must start with one of these
   - `action_verbs_rejected` — reject objectives starting with these

3. Read `publishing-house/spec.yaml` for project state and pre-populated fields
4. Read design template at `@rhdp-publishing-house/skills/intake/references/design-template.md`
5. Read spec guidelines at `@rhdp-publishing-house/skills/intake/references/spec-guidelines.md`
6. Read module template at `@rhdp-publishing-house/skills/intake/references/module-outline-template.md`

## Dispatch

After pre-flight, follow the intake procedures:

1. Follow `procedures/02-interview.md`
2. Follow `procedures/03-design-doc.md`
3. Follow `procedures/04-module-outlines.md`
4. If RCARS vetting results exist → follow `procedures/05-spec-refinement.md`
5. Follow `procedures/06-approval-and-submit.md`

After `06-approval-and-submit.md` completes, **return to the orchestrator** (if dispatched) or **STOP** (if invoked directly).

## Pre-populated Fields

Before asking intake questions, check spec.yaml for fields already set by the
RHDH template or orchestrator:
- `project.slug` — project identifier
- `project.owner_email` — author email
- `project.content_type` — lab, demo, workshop, onboarding
- `project.deployment_mode` — rhdp_published or self_published
- `project.initiative_key` — e.g., rh1_2027
- `project.showroom_type` — classic or zero_touch

**Skip asking about any field that already has a value.**

## Key Behavioral Notes

- Push back on vague objectives
- Propose module structures and validate them
- Identify gaps the user hasn't thought of
- Scale question depth to project complexity

**Goal: Rigorous exploration through conversation, not just filling in a template.**
