# siab-site-orchestrator — Claude Code conventions

You are operating the **sitegen** workflow: producing one-pager / multi-page Astro landing
pages for clients. Org is `optidigi`. New site repos: `site-<slug>`, public, images on `ghcr.io`.

## Repos in this workspace

You operate from the orchestrator root (the directory holding this `CLAUDE.md`). Wherever the operator cloned this repo — any path is fine — the sibling clones live as gitignored child dirs alongside it.

- `./` — this orchestrator. Holds CLAUDE.md, preflight.md, prompt.md, .claude/.
- `./siab-site-template/` — Astro boilerplate (cloned, gitignored). Source for new sites.
- `./siab-site-themes/` — theme building blocks under `astro/<name>/` and `plain/<name>/` (cloned, gitignored).
- `./site-<slug>/` — ephemeral working copy per engagement. Created from template, deleted after deploy.

## Downstream CMS-ification contract

Sites produced by this orchestrator are designed to be cleanly CMS-ified by `siab-payload-orchestrator`. To stay compatible:

- **Phase 2 always generates `siteManifest.json`** at site repo root (per-tenant manifest read by payload-seeder during CMS-ification). Template ships `siteManifest.example.json` as the source.
- **Phase 3 maps theme tokens into the full role-token set** — `--font-{title,heading,text,script,serif,sans}` + `--radius-{sm,md,lg}` — not just colors. The template's `src/components/cms/*` renderers consume these via inline `style` props for the post-CMS-ification rendering surface.
- **The template ships `src/components/cms/*` renderers + `RtNodeRenderer`** for the post-CMS-ification surface. These are inert in static-mode (only invoked via the wave-3 preview-iframe) and become live when `site-converter` (in `siab-payload-orchestrator`) rewires page routes during Phase 5 of CMS-ification.

Reviewer Phase 7 gates on `siteManifest.json` presence — sites missing it can't ship.

## Subagents available

- `copywriter` — page copy + meta with SEO/tone awareness. Dispatch in Phase 4.
- `auditor` — Lighthouse + axe + header checks. Dispatch in Phase 6.
- `reviewer` — pre-push final-gate review. Dispatch in Phase 7. Uses `code-reviewer` agent type.

See `.claude/agents/*.md` for input/output contracts.

## Hard rules

1. Read `preflight.md` first when starting a new site. Summarize back what you understood. Wait for user confirmation.
2. Then read `prompt.md` and run the 10-phase runbook.
3. Never modify `siab-site-template/` or `siab-site-themes/` during a run. Only write into `site-<slug>/`.
4. Never push to `main` of `site-<slug>` until the user has approved the preview (Phase 8 gate).
5. Never strip the SEO baseline (sitemap, robots.txt, llms.txt, JSON-LD `Organization`, security headers).
6. For non-trivial copy, dispatch the `copywriter` subagent — don't write copy inline.
7. VPS-side compose / nginx vhost / TLS / DNS are out of scope. Help diagnose if asked, don't SSH.

## Quality floors (sites must meet these before sign-off)

| Metric | Floor |
| --- | --- |
| Lighthouse Performance (mobile) | ≥75 |
| Lighthouse Accessibility | ≥85 |
| Lighthouse Best Practices | ≥85 |
| Lighthouse SEO | ≥95 |
| axe-core | no critical / serious violations |
| Security headers | CSP, X-Frame-Options, X-Content-Type-Options present |

See the auditor's report and address must-fix items before requesting user sign-off.

## Re-engagements

If client wants changes weeks later, `git clone optidigi/site-<slug>` back into the orchestrator root as `./site-<slug>/`, edit, push. Skip Phases 1–3, jump to whichever phase applies.
