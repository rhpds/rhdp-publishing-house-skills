---
name: rhdp-publishing-house:config-helper
description: This skill should be used when the user asks to "set up showroom", "configure showroom tabs", "create site.yml", "set up ui-config.yml", "scaffold the showroom structure", "add a tab", "change the theme", or "switch to zerotouch".
context: main
---

# Showroom Config Helper

You help authors set up and configure Showroom content repositories. You handle the
first decision (showroom vs zerotouch content mode), generate the correct config files,
configure tabs, and create automation directory skeletons.

See @rhdp-publishing-house/skills/config-helper/references/showroom-patterns.md for the three lab patterns and their complete config examples.
See @rhdp-publishing-house/skills/config-helper/references/config-files.md for the full reference on every config file.

## Step 1 — Detect Repo Context

Check these files and directories silently to understand the current state:

| Check | Meaning |
|-------|---------|
| `publishing-house/spec.yaml` exists | This is a Publishing House project |
| `_scaffolds/` directory exists | PH project that has NOT been scaffolded yet |
| `site.yml` exists | Antora playbook already configured |
| `ui-config.yml` exists | Showroom UI already configured |
| `content/antora.yml` exists | Antora component present |
| `runtime-automation/` exists | Zerotouch automation skeleton in place |
| `config/` directory exists | ZT Guided infrastructure config present |

## Step 2 — Route Based on State

### Route A: PH project with `_scaffolds/` still present

The project has not been scaffolded. Read the intake spec to determine the pattern, then delegate to `scaffold.py`.

**A.1 — Read spec.yaml to determine pattern:**

Read `publishing-house/spec.yaml` and extract `project.showroom_type`. Map to scaffold pattern:

| `project.showroom_type` | scaffold.py `--pattern` |
|--------------------------|-------------------------|
| `classic` | `agd-open` |
| `zero_touch` | `zt-guided` |
| `guided` | `agd-guided` (not currently offered during intake) |

If `project.showroom_type` is empty, unset, or unrecognised, fall back to asking:
> I couldn't determine the showroom type from your spec. Which lab pattern do you want?
>
> 1. **Open** (`agd-open`) — self-paced, free-form navigation
> 2. **Guided** (`agd-guided`) — sequential modules with solve/validate buttons
> 3. **ZT Guided** (`zt-guided`) — guided + Project Zero infrastructure

**A.2 — Confirm and run scaffold.py:**

Confirm the detected pattern with the user before running:
> Your spec has `showroom_type: <value>` — I'll scaffold as **<pattern description>**. Proceed?

Then run:
```bash
python scaffold.py --pattern <pattern-name> --force
```

**A.3 — Verify and adjust:**

After scaffold.py completes, proceed to Route C (modification flow) to verify and adjust the generated config.

### Route B: No config files exist (new repo)

No `site.yml` or `ui-config.yml` found. Run the full setup flow.

**B.1 — Choose content mode:**

> First decision: how should learners navigate your lab?
>
> 1. **Showroom** (`type: showroom`) — free-form browse with table of contents. Best for self-paced workshops, demos, and reference environments.
> 2. **Zerotouch** (`type: zerotouch`) — guided linear progression with Next/Previous, Solve, and Validate buttons. Best for instructor-led labs requiring sequential completion.

**B.2 — Choose infrastructure type:**

> What infrastructure will your lab run on?
>
> 1. **OCP** — OpenShift cluster (AgnosticD v2)
> 2. **VM** — Virtual machines with bastion (AgnosticD v2)
> 3. **ZT** — Project Zero (zero-touch VMs with custom networking)

ZT always uses zerotouch content mode. If the user chose showroom mode and ZT infrastructure, confirm:
> ZT infrastructure typically uses zerotouch mode for solve/validate automation. Continue with showroom mode, or switch to zerotouch?

**B.3 — Ask for lab title:**

> What is the title for your lab?

**B.4 — Generate config files:**

Based on the content mode and infrastructure type, determine the pattern (Open, Guided, or ZT Guided) and generate all files. Use the exact templates from @rhdp-publishing-house/skills/config-helper/references/showroom-patterns.md — do not improvise config structure.

Files to generate:

1. **`site.yml`** — set title, choose the correct theme bundle, register extensions. For zerotouch, include the dev-mode extension.

2. **`ui-config.yml`** — use the correct format for the content mode:
   - Showroom: `type: showroom` with `view_switcher` and `tabs`
   - Zerotouch: `antora:` block with module labels and `tabs`
   - Configure tabs based on infrastructure:
     - OCP: OCP Console tab + terminal tab (`path: /wetty`, `port: 443`)
     - VM: stacked terminal tab (Bastion + Worker)
     - ZT: terminal tab (`url: /wetty`)

3. **`content/antora.yml`** — set title, nav path, default attributes for the infrastructure type

4. **`content/modules/ROOT/nav.adoc`** — create with index.adoc entry

5. **`content/modules/ROOT/pages/index.adoc`** — create a minimal stub

6. **`podman-compose.yaml`** — create from the standard template in the config-files reference

7. If zerotouch:
   - **`runtime-automation/module-01/`** — create `setup.yml`, `solve.yml`, `validate.yml` stubs

8. If ZT Guided:
   - **`config/instances.yaml`** — default RHEL 9.5 bastion VM
   - **`config/networks.yaml`** — default + secondary networks
   - **`config/firewall.yaml`** — egress TCP 443
   - **`setup-automation/setup.yml`** — environment setup stub

**B.5 — Summarize what was created:**

Show a tree of generated files and explain the key next steps:
- Edit `content/antora.yml` to set your lab title and runtime attribute defaults
- Replace placeholder tabs in `ui-config.yml` with real service URLs
- Start writing content in `content/modules/ROOT/pages/`
- Preview locally with `podman-compose up`

### Route C: Config files exist (modification flow)

The repo already has `site.yml` and/or `ui-config.yml`.

1. Read existing config files to detect the current pattern. Use the detection rules from @rhdp-publishing-house/skills/config-helper/references/showroom-patterns.md.

2. Understand what the user wants to change. Common requests:
   - **Add a tab** — add entry to `tabs:` list in ui-config.yml
   - **Remove a tab** — remove entry from `tabs:` list
   - **Change tab order** — reorder entries
   - **Adjust layout width** — change `default_width` (showroom mode only)
   - **Change title** — update `site.title` in site.yml and/or `title` in antora.yml
   - **Add antora.yml attributes** — add to `asciidoc.attributes`
   - **Switch content mode** — change theme in site.yml AND format of ui-config.yml (significant change — confirm with user)
   - **Add a module** — create page file, update nav.adoc, and if zerotouch update `antora.modules` in ui-config.yml and create runtime-automation stubs
   - **Replace placeholders** — swap `/placeholder` tabs with real URLs

3. Apply changes while preserving format consistency. When modifying ui-config.yml, maintain the correct format for the detected content mode.

4. After modifications, suggest running `rhdp-publishing-house:config-reviewer` to validate the result.

## Rules

- Always use the exact config structures from the reference files — do not invent new YAML keys
- When generating zerotouch ui-config.yml, the `antora.modules` entries must match the page filenames in `content/modules/ROOT/pages/`
- Tab URLs that reference cluster services should use `${DOMAIN}` variable substitution (not `{DOMAIN}`)
- Never modify `site.yml` content source settings — `start_path: content` and `url: .` are fixed
- For PH projects, do not modify `publishing-house/` directory contents — that is PH skill territory
- When switching content modes, both `site.yml` (bundle URL) and `ui-config.yml` (format) must change together
