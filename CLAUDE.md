# sitegen-orchestrator — Claude Code conventions

You are operating the **sitegen** workflow: producing one-pager / multi-page Astro landing
pages for clients. Org is `optidigi`. New site repos: `site-<slug>`, public, images on `ghcr.io`.

## Repos in this workspace

You operate from the orchestrator root (the directory holding this `CLAUDE.md`). Wherever the operator cloned this repo — any path is fine — the sibling clones live as gitignored child dirs alongside it.

- `./` — this orchestrator. Holds CLAUDE.md, preflight.md, prompt.md, .claude/.
- `./sitegen-template/` — Astro boilerplate (cloned, gitignored). Source for new sites.
- `./sitegen-themes/` — theme building blocks under `astro/<name>/` and `plain/<name>/` (cloned, gitignored).
- `./site-<slug>/` — ephemeral working copy per engagement. Created from template, deleted after deploy.

## Subagents available

- `copywriter` — page copy + meta with SEO/tone awareness. Dispatch in Phase 4.
- `auditor` — Lighthouse + axe + header checks. Dispatch in Phase 6.
- `reviewer` — pre-push final-gate review. Dispatch in Phase 7. Uses `code-reviewer` agent type.

See `.claude/agents/*.md` for input/output contracts.

## Hard rules

1. Read `preflight.md` first when starting a new site. Summarize back what you understood. Wait for user confirmation.
2. Then read `prompt.md` and run the 10-phase runbook.
3. Never modify `sitegen-template/` or `sitegen-themes/` during a run. Only write into `site-<slug>/`.
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
