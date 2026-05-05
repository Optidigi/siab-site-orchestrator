# siab-site-orchestrator

Workflow for spinning up cheap, quick, quality Astro landing pages under `optidigi/site-<slug>` with images on `ghcr.io`.

## Setup (fresh machine)

Clone the orchestrator wherever you want it — any path works (`~/Code/sitegen/`, `~/projects/`, your Desktop, anywhere). The two sibling repos are cloned inside it; the orchestrator's `.gitignore` ignores them automatically.

```bash
git clone git@github.com:Optidigi/siab-site-orchestrator.git
cd siab-site-orchestrator
git clone git@github.com:Optidigi/siab-site-template.git
git clone git@github.com:Optidigi/siab-site-themes.git
```

Run Claude Code from inside `siab-site-orchestrator/`.

## Run a new site

Tell Claude `/new-site` (or just "let's start a new site"). The agent reads `preflight.md`, asks you to confirm understanding, then reads `prompt.md` and walks the 10-phase runbook.
