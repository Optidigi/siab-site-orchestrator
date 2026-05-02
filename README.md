# sitegen-orchestrator

Workflow for spinning up cheap, quick, quality Astro landing pages under `optidigi/site-<slug>` with images on `ghcr.io`.

See `docs/superpowers/specs/2026-05-02-sitegen-orchestrator-design.md` for the design.

## Setup (fresh machine)

```bash
git clone git@github.com:optidigi/sitegen-orchestrator.git ~/Desktop/env/sandbox
cd ~/Desktop/env/sandbox
git clone git@github.com:optidigi/sitegen-template.git
git clone git@github.com:optidigi/sitegen-themes.git
```

Run Claude Code from this directory.

## Run a new site

Tell Claude `/new-site` (or just "let's start a new site"). The agent reads `preflight.md`, asks you to confirm understanding, then reads `prompt.md` and walks the 10-phase runbook.
