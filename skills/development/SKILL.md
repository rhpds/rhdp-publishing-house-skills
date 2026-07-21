---
name: rhdp-publishing-house:development
description: This skill should be used when the user asks to "write a module", "draft content", "start writing", "edit my content", "review the modules", "build automation", "create the catalog", or "what's next to develop". Handles writing, editing, and automation during the development stage.
---

---
context: main
model: claude-opus-4-6
---

# Development Agent

**RULE: If any `publishing-house/tools/` script exits with a non-zero exit code, STOP immediately.** Show the error output to the author and say there was an issue calling the backend. Do not continue the skill.

You handle the development phase of the Publishing House lifecycle. This skill is
self-sufficient — it works whether dispatched by the orchestrator or invoked directly.

## Tool Boundaries

**Do NOT use** Central API tools directly. You work locally: read files, write content, update spec.yaml.

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

  **STOP.**
- `no-spec` → show: "`publishing-house/spec.yaml` is missing." **STOP.**

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

Extract `project_id`. If empty → **STOP.**

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

- Has `cred:` and `central:` → proceed.
- `no-central` → show: "Cannot find Central API URL. Check that `catalog-info.yaml` has a **Central** link." **STOP.**
- `no-cred` → show the portal URL and ask for the key (same as orchestrator auth flow). **Wait for key, save, then proceed.**

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

If stage is not `development` → show:
> Cannot start this skill because the project is in **{stage}** stage. This skill requires **development**.

**STOP — do not proceed.**

### Step 5 — Read project context

1. Read `publishing-house/spec.yaml` for project metadata and spec data
2. Read `publishing-house/spec/design.md` for the design spec
3. Read module outlines in `publishing-house/spec/modules/`

## Dispatch

Based on what the user asked for:

- **"write module N"** / **"start writing"** / **"write all"** → follow `procedures/writer.md`
- **"edit module N"** / **"review content"** / **"technical edit"** → follow `procedures/editor.md`
- **"build automation"** / **"create the catalog"** / **"write the AgnosticV config"** → follow `procedures/automation.md`
- **No specific request** / **"what's next"** → show development dashboard:

### Development Dashboard

Present the current state:

> **Development Status**
>
> **Modules:**
> [For each module outline, check if a corresponding .adoc exists in content/modules/ROOT/pages/]
> - Module 1: [title] — [written / not started]
> - Module 2: [title] — [written / not started]
>
> **Automation:** [done / not started]
>
> What would you like to work on?

## Rules

- Never tell the author to run any script
- The development skill dispatches to procedures but does not own workflow advancement
- Each procedure handles its own commit and push
