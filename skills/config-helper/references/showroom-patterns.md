# Showroom Lab Patterns Reference

## Overview

Two content modes set via `type` in `ui-config.yml`:

- `type: showroom` — free-form browse, rhdp_showroom_theme, TOC + next/prev in content pane
- `type: zerotouch` — guided linear, nookbag-bundle, Nookbag handles navigation externally (progress bar, Next/Previous, Solve, Skip, Exit)

Theme MUST match content mode. Mismatch produces duplicate or missing navigation.

---

## Pattern 1: Open (classic)

- `type: showroom`
- Theme: `rhdp_showroom_theme`
- site.yml bundle: `https://github.com/rhpds/rhdp_showroom_theme/releases/download/v2.0.3/ui-bundle.zip` (or `latest`)
- ui-config.yml uses `type: showroom` header with `default_width`, `persist_url_state`, `view_switcher`
- No `antora:` module labels block
- No `runtime-automation/` directory
- Best for: self-paced workshops, demos, reference environments

### Infrastructure variants

**Open OCP**: tabs include OCP Console (`url: 'https://console-openshift-console.${DOMAIN}'`), terminal (`path: /wetty`)

**Open VM**: tabs include stacked terminals (primary Bastion `path: /wetty` + secondary Worker `secondary_path: /terminal2`)

### Complete Open OCP ui-config.yml

```yaml
---
type: showroom

default_width: 30
persist_url_state: true

view_switcher:
  enabled: true
  default_mode: split

tabs:
  - name: OCP Console
    url: 'https://console-openshift-console.${DOMAIN}'
  - name: Bastion ${USER}
    path: /wetty
  - name: Placeholder
    url: /placeholder
```

### Complete Open VM ui-config.yml

```yaml
---
type: showroom

default_width: 30
persist_url_state: true

view_switcher:
  enabled: true
  default_mode: split

tabs:
  - name: Bastion
    path: /wetty
    secondary_name: Worker
    secondary_path: /terminal2
  - name: Placeholder
    url: /placeholder
```

### Open site.yml (same for both OCP and VM)

```yaml
---
site:
  title: Showroom Template Demo
  url: https://github.com/rhpds/showroom-template
  start_page: modules::index.adoc
content:
  sources:
    - url: .
      start_path: content
ui:
  bundle:
    url: https://github.com/rhpds/rhdp_showroom_theme/releases/download/v2.0.3/ui-bundle.zip
    snapshot: true
antora:
  extensions:
    - require: '@sntke/antora-mermaid-extension'
      mermaid_library_url: https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs
      script_stem: header-scripts
      mermaid_initialize_options:
        start_on_load: true
    - require: '@andrew-jones/antora-tabs-extension'
output:
  dir: ./www
```

---

## Pattern 2: Guided (AgD v2)

- `type: zerotouch`
- Theme: `nookbag-bundle`
- site.yml bundle: `https://github.com/rhpds/nookbag-bundle/releases/download/v0.0.3/ui-bundle.zip`
- site.yml includes dev-mode extension: `- require: /antora/lib/dev-mode.js` with `enabled: true`
- ui-config.yml uses `antora:` block with module name/label mappings (drives solve/validate button UI)
- Requires `runtime-automation/` directory with `solve.yml` + `validate.yml` per module (optional `setup.yml` for module-01)
- Best for: instructor-led or structured labs requiring sequential completion and validation

### Infrastructure variants

**Guided OCP**: tabs include OCP Console + terminal (`path: /wetty`)

**Guided VM**: tabs include stacked terminals (Bastion `path: /wetty` + Worker `secondary_path: /terminal2`)

### Complete Guided OCP ui-config.yml

```yaml
---
antora:
  name: modules
  dir: www
  modules:
    - name: index
      label: "Introduction"
    - name: module-01
      label: "Module 1: Your First Lab Page"
    - name: module-02
      label: "Module 2: Another Module"

tabs:
  - name: OCP Console
    url: 'https://console-openshift-console.${DOMAIN}'
  - name: ">_ terminal"
    path: /wetty
  - name: Placeholder
    url: /placeholder
```

### Complete Guided VM ui-config.yml

```yaml
---
antora:
  name: modules
  dir: www
  modules:
    - name: index
      label: "Introduction"
    - name: module-01
      label: "Module 1: Your First Lab Page"
    - name: module-02
      label: "Module 2: Another Module"

tabs:
  - name: ">_ Bastion"
    path: /wetty
    secondary_name: Worker
    secondary_path: /terminal2
  - name: Placeholder
    url: /placeholder
```

### Guided site.yml (same for both OCP and VM)

```yaml
---
site:
  title: Showroom Template Demo
  url: https://github.com/rhpds/showroom-template
  start_page: modules::index.adoc
content:
  sources:
    - url: .
      start_path: content
ui:
  bundle:
    url: https://github.com/rhpds/nookbag-bundle/releases/download/v0.0.3/ui-bundle.zip
    snapshot: true
antora:
  extensions:
    - require: '@sntke/antora-mermaid-extension'
      mermaid_library_url: https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs
      script_stem: header-scripts
      mermaid_initialize_options:
        start_on_load: true
    - require: '@andrew-jones/antora-tabs-extension'
    - require: /antora/lib/dev-mode.js
      enabled: true
output:
  dir: ./www
```

### Guided runtime-automation structure

```
runtime-automation/
├── module-01/
│   ├── setup.yml      # optional, runs once before the module
│   ├── solve.yml      # runs when user clicks Solve
│   └── validate.yml   # runs when user clicks Validate
└── module-02/
    ├── solve.yml
    └── validate.yml
```

The `antora.modules` entries in ui-config.yml must match the page filenames in `content/modules/ROOT/pages/`. The `name` field is the filename without `.adoc`, and the `label` is the display name in the Nookbag UI.

---

## Pattern 3: ZT Guided (Zero Touch)

- Same as Guided but with additional infrastructure config
- `type: zerotouch`, nookbag-bundle theme
- Same site.yml as Guided
- ui-config.yml same format as Guided but terminal tab uses `url: /wetty` (NOT `path` + `port` — ZT networking differs)
- Additional directories beyond Guided:
  - `config/instances.yaml` — VM definitions (image, memory, cores, tags, networks)
  - `config/networks.yaml` — network definitions
  - `config/firewall.yaml` — egress rules
  - `setup-automation/setup.yml` — environment setup playbook (runs once on pod creation)
- Best for: labs running on Project Zero infrastructure

### Complete ZT ui-config.yml

```yaml
---
antora:
  name: modules
  dir: www
  modules:
    - name: index
      label: "Introduction"
    - name: module-01
      label: "Module 1: Your First Lab Page"
    - name: module-02
      label: "Module 2: Another Module"

tabs:
  - name: ">_ terminal"
    url: /wetty
  - name: Placeholder
    url: /placeholder
```

### ZT config directory examples

```yaml
# config/instances.yaml
---
containers: []
virtualmachines:
  - name: "host1"
    image: "rhel-9.5"
    memory: "2G"
    cores: 1
    image_size: "40G"
    tags:
      - key: "AnsibleGroup"
        value: "bastions"
    networks:
      - default
      - secondary
```

```yaml
# config/networks.yaml
---
- name: default
- name: secondary
```

```yaml
# config/firewall.yaml
---
egress:
  - ports:
      - protocol: TCP
        port: 443
```

---

## Pattern detection rules

How to detect which pattern is in use from existing config:

1. If ui-config.yml has `type: showroom` → Open pattern
2. If ui-config.yml has `antora:` block (with or without `type: zerotouch`) → Zerotouch pattern (Guided or ZT)
3. If zerotouch AND `config/` directory exists → ZT Guided
4. If zerotouch AND no `config/` directory → Guided (AgD v2)
5. Bundle URL in site.yml confirms: `rhdp_showroom_theme` = Open, `nookbag-bundle` = Zerotouch

---

## Publishing House integration

When a PH project has `_scaffolds/` directory and `scaffold.py`:

- The intake writes `project.showroom_type` to `publishing-house/spec.yaml` during Phase 1 (discovery)
- config-helper reads this field and maps it to the scaffold pattern:

| `project.showroom_type` | scaffold.py `--pattern` | Content mode | Theme |
|--------------------------|-------------------------|--------------|-------|
| `classic` | `agd-open` | `type: showroom` | rhdp_showroom_theme |
| `zero_touch` | `zt-guided` | `type: zerotouch` | nookbag-bundle |
| `guided` | `agd-guided` | `type: zerotouch` | nookbag-bundle |

Note: `agd-guided` is not currently offered during intake. Only `classic` and `zero_touch` are selectable.

- config-helper confirms the detected pattern with the user, then runs `scaffold.py --pattern <name> --force`
- scaffold.py copies pattern-specific stubs, sets `showroom_type` and `infrastructure` in spec.yaml, deletes `_scaffolds/`
- After scaffold.py runs, config-helper operates in modification mode on the generated files

---

## Common antora.yml (shared across all patterns)

```yaml
name: modules
title: My Lab Title
version: ~
nav:
  - modules/ROOT/nav.adoc

asciidoc:
  attributes:
    experimental: true
    page-pagination: true
    guid: '%GUID%'
    ssh_user: lab-user
    ssh_password: changeme
    openshift_console_url: 'https://console-openshift-console.apps.cluster-%GUID%.example.com'
    openshift_cluster_ingress_domain: 'apps.cluster-%GUID%.example.com'
```

---

## Minimum directory layout for all patterns

```
your-content-repo/
├── content/
│   ├── antora.yml
│   └── modules/ROOT/
│       ├── nav.adoc
│       └── pages/
│           ├── index.adoc
│           └── module-01.adoc
├── site.yml
└── ui-config.yml
```
