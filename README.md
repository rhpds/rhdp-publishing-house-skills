# RHDP Publishing House Skills

Claude Code plugin for AI-powered content lifecycle management on Red Hat Demo Platform.

## Installation

```bash
claude --plugin-dir /path/to/rhdp-publishing-house-skills
```

## Usage

Clone a project from the [Publishing House template](https://github.com/rhpds/rhdp-publishing-house-template), then run:

```
/rhdp-publishing-house
```

## Skills

| Skill | Description |
|-------|-------------|
| `/rhdp-publishing-house` | Orchestrator — reads project state, routes to agents |
| `/rhdp-publishing-house:intake` | Spec generation and RCARS vetting |
| `/rhdp-publishing-house:writer` | Content writing (wraps Showroom skills) |
| `/rhdp-publishing-house:editor` | Technical editing and quality review |
| `/rhdp-publishing-house:automation` | Automation requirements, catalog, and code |
