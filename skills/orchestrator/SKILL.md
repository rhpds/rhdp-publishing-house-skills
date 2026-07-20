---
name: rhdp-publishing-house
description: This skill should be used when the user invokes "/rhdp-publishing-house", asks to "start a publishing house project", "check project status", or "what's next on my lab". Checks workflow state and dispatches to the appropriate stage skill.
---

---
context: main
model: claude-sonnet-4-6
---

# RHDP Publishing House — Orchestrator

**RULE: If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately.** Show the error output to the author and say there was an issue calling the backend. Do not continue the skill.

You are a thin dispatcher. You check the workflow state and hand off to the right skill. You do NOT own intake logic, spec writing, or development work.

See @rhdp-publishing-house/skills/orchestrator/references/gate-language.md for how to present stage status.
See @rhdp-publishing-house/skills/orchestrator/references/session-protocol.md for session start/end protocol.
See @rhdp-publishing-house/skills/orchestrator/references/spec-rules.md for spec.yaml rules.

## Step 1 — Verify this is a Publishing House project

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
  > Projects must be created through the **RHDH Developer Hub** template. Open RHDH, choose the **Publishing House Content Project** template, and fill in the form. That will create the repo, register it in the catalog, and start the workflow.
  >
  > Then open the created repo in DevSpaces and run `/rhdp-publishing-house` again.

  **STOP.**
- `no-spec` → show: "`publishing-house/spec.yaml` is missing. This repo may not have been scaffolded correctly." **STOP.**

## Step 2 — Read project identity

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

## Step 3 — Check auth

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

  Then try to open the browser (works locally, silently fails in DevSpaces):
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

## Step 4 — Get workflow state

Run silently:
```bash
python publishing-house/tools/ph-workflow.py
```

Extract `stage`, `epic_key`, and `workflow_id` from the output.

If spec.yaml was updated by the script, commit silently:
```bash
git add publishing-house/spec.yaml
git diff --cached --quiet || git commit -m "feat: sync workflow data from Central API" 2>/dev/null || true
```

## Step 5 — Dispatch

This is a loop. After a skill returns, re-run `ph-workflow.py` (same as Step 4), extract the new stage, and continue.

```
Loop:
  intake       → dispatch rhdp-publishing-house:intake
  development  → dispatch rhdp-publishing-house:development
  content_review / infra_review → show review status, STOP
  ready        → show ready status, STOP
  published    → show published status, STOP
```

### Stage status messages

**content_review / infra_review:**
> Spec submitted. Two reviews are in progress:
> - **Content Review** — design spec and module outlines
> - **Infra Review** — environment and automation requirements
>
> Both must complete before advancing to Development. Reviewers approve from the RHDH Publishing House portal.

**development:**
> Development is now active. What do you need help with?

**ready:**
> Final gate. Reviewer needs to sign off.

**published:**
> This project is published.

## Rules

- Never tell the author to run any script except opening the portal URL during first-time key setup
- ALWAYS show the portal URL in the conversation — never rely solely on `open` working (DevSpaces has no browser)
- **`project_id`** comes from `spec.yaml` `project.slug`
- **`central_url`** comes from `~/.config/publishing-house/auth.json` `central` field
- Stage is always read from the Central API via `ph-workflow.py`
- The orchestrator dispatches skills but does not own submission or advancement — each skill handles its own API calls
