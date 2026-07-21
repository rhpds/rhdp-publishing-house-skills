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
import json, os, yaml
f = os.path.expanduser('~/.config/publishing-house/auth.json')
if os.path.exists(f):
    d = json.load(open(f))
    cred = d.get('credential', '')
    central = d.get('central', '')
    print(f'cred:{cred[:8]}' if cred else 'no-cred')
    print(f'central:{central}')
else:
    central = ''
    try:
        ci = yaml.safe_load(open('catalog-info.yaml'))
        for link in ci.get('metadata', {}).get('links', []):
            if link.get('title') == 'Central':
                central = link['url']
                break
    except Exception:
        pass
    if central:
        os.makedirs(os.path.dirname(f), exist_ok=True)
        with open(f, 'w') as fh:
            json.dump({'central': central}, fh, indent=2)
        os.chmod(f, 0o600)
        print('no-cred')
        print(f'central:{central}')
    else:
        print('no-central')
"
```

Extract `central_url` from the `central:` line.

- Has `cred:` and `central:` → proceed.
- `no-central` → show: "Cannot find Central API URL. Check that `catalog-info.yaml` has a **Central** link." **STOP.**
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

### Step 4 — Sync workflow data

Run sync to fetch current workflow state, persist workflow_id/epic_key if not already set, and pull any rejections:
```bash
python publishing-house/tools/ph-sync.py
```
Extract `stage`, `workflow_id`, `epic_key`, and `unresolved_rejections` from the output. Commit any changes:
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

After pre-flight, check the `unresolved_rejections` count from `ph-sync.py` output:

**If `unresolved_rejections` > 0:**
1. Follow `procedures/01-rejection-handler.md` — address unresolved feedback first
2. The rejection handler determines the re-entry point (module outlines or submit)

**If `unresolved_rejections` is 0 (fresh intake or all rejections resolved):**
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
