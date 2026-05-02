# Sitegen Orchestrator — Design

**Date:** 2026-05-02
**Status:** Approved (pre-implementation)
**Owner:** admin@optidigi.nl

## Purpose

Define a repeatable workflow for producing cheap, quick, quality landing pages
(one-pager or up to ~5-page) for clients. Each engagement produces a static
Astro site, packaged as a Docker image, published to GitHub Container Registry
under the `optidigi` org, and pulled onto a VPS server-side via
`docker compose`.

The workflow is operated by Claude Code in a sandbox directory and is
controlled by a small set of static prompt files plus three subagent
specialists.

Quality bar: above-average (Lighthouse mobile ≥75 perf, ≥85 a11y, ≥85 BP, ≥95
SEO; no critical/serious axe violations; full SEO baseline; security headers).

## Architecture — four GitHub repositories

All repositories live under the GitHub organization `optidigi` and are public
by default.

```
optidigi/sitegen-orchestrator   → cloned to ~/Desktop/env/sandbox/
                                  Holds CLAUDE.md, preflight.md, prompt.md,
                                  .mcp.json, .claude/, .gitignore, README.md.
                                  Pure workflow — never built or deployed.

optidigi/sitegen-template       → cloned to sandbox/sitegen-template/
                                  Astro 5.x + Tailwind boilerplate with full
                                  SEO baseline, Dockerfile (static→nginx:alpine),
                                  GHA that publishes to ghcr.io on push to main.

optidigi/sitegen-themes         → cloned to sandbox/sitegen-themes/
                                  Subdirs astro/<name>/ and plain/<name>/.
                                  Curated favorites + ad-hoc additions.

optidigi/site-<slug>            → CREATED PER ENGAGEMENT.
                                  Built in sandbox/site-<slug>/, pushed when
                                  user approves preview, working dir deleted
                                  after the user confirms VPS deploy.
                                  Inherits Dockerfile + GHA from template.
```

`<slug>` is a short canonical handle (e.g. `amicare` for `ami-care.nl`,
`amicare.nl`, `a-m-i-care.com`).

## Sandbox layout

```
sandbox/
├── CLAUDE.md                 # auto-loaded, high-level conventions
├── preflight.md              # background context, agent reads on demand
├── prompt.md                 # runbook, agent reads after preflight confirmed
├── README.md                 # human-facing setup instructions
├── .mcp.json                 # project-scoped MCP config (currently minimal)
├── .claude/
│   ├── settings.json         # permissions allowlist
│   ├── agents/
│   │   ├── copywriter.md
│   │   ├── auditor.md
│   │   └── reviewer.md
│   └── commands/
│       └── new-site.md       # optional /new-site slash command
├── .gitignore                # ignores sitegen-template/, sitegen-themes/, site-*/
├── docs/                     # specs, plans, notes (this file lives here)
├── sitegen-template/         # cloned, gitignored from orchestrator
├── sitegen-themes/           # cloned, gitignored from orchestrator
└── site-<slug>/              # ephemeral, gitignored, deleted after deploy
```

### Repo nesting strategy: plain ignored clones

The three sibling directories are independent git repositories. The
orchestrator's `.gitignore` excludes them, so `git status` in `sandbox/`
treats them as if they don't exist. Reproducibility on a fresh machine is
handled by the orchestrator's `README.md`, which lists the three `git clone`
commands required.

This is preferred over git submodules because `sitegen-template` and
`sitegen-themes` evolve independently of the orchestrator at different
cadences; submodules would force a parent commit for every theme tweak with
no offsetting benefit (we do not need atomic-pinned reproducibility).

## `sitegen-template` (the boilerplate)

**Stack**

- Astro 5.x, output `static`
- Tailwind CSS 4 via `@tailwindcss/vite`
- `@astrojs/sitemap`, `@astrojs/check`, `astro-seo`
- `sharp` (built-in image pipeline)
- pnpm

**Directory shape**

```
sitegen-template/
├── src/
│   ├── components/
│   │   ├── seo/Seo.astro
│   │   ├── seo/JsonLdOrganization.astro
│   │   ├── seo/JsonLdLocalBusiness.astro
│   │   └── forms/ContactForm.astro
│   ├── content/
│   │   ├── config.ts                  # zod schemas
│   │   ├── pages/                     # markdown per page
│   │   └── site.ts                    # nav, footer, NAP, socials
│   ├── layouts/BaseLayout.astro
│   ├── pages/
│   │   ├── index.astro
│   │   ├── 404.astro
│   │   └── robots.txt.ts
│   └── styles/global.css
├── public/
│   ├── llms.txt
│   ├── humans.txt
│   ├── .well-known/security.txt
│   ├── favicon.ico, apple-touch-icon.png, manifest.json
│   └── og-default.png
├── Dockerfile
├── .dockerignore
├── nginx.conf
├── .github/workflows/publish.yml
├── astro.config.mjs
├── tailwind.config.ts
├── tsconfig.json, package.json, pnpm-lock.yaml
└── README.md
```

**Conventions baked into the template**

- `astro.config.mjs` reads `SITE_URL` from env and uses it for canonical / OG.
- `src/content/site.ts` holds site-wide data (NAP, socials, hours) and is
  consumed by JSON-LD components and the footer.
- `BaseLayout.astro` automatically wires `Seo.astro`; pages just supply
  `{title, description, ogImage}`.
- `ContactForm.astro` renders a Web3Forms POST form when
  `PUBLIC_WEB3FORMS_KEY` is set; otherwise falls back to `mailto:`.
- `nginx.conf` ships with: gzip, 1y cache for hashed assets, no-cache for
  HTML, security headers (CSP, HSTS, X-Frame-Options, X-Content-Type-Options,
  Referrer-Policy, Permissions-Policy).

**Dockerfile (static → nginx:alpine)**

Multi-stage: `node:lts-alpine` for `pnpm install` + `pnpm build`, then
`nginx:alpine` serving `/usr/share/nginx/html` with `nginx.conf` mounted at
`/etc/nginx/conf.d/default.conf`.

**GitHub Actions: `.github/workflows/publish.yml`**

- **Trigger:** push to `main`
- **Auth:** uses `GITHUB_TOKEN` to log in to `ghcr.io` (zero secrets needed
  for public repos)
- **Tags:** `latest` plus `sha-<short>`, generated via `docker/metadata-action`
- **Steps:** checkout → buildx → login → metadata → build-push

VPS pulls from `ghcr.io/optidigi/site-<slug>:latest`; rollback path is to pin
`sha-<short>` in the VPS compose file.

## SEO baseline (always on)

Generated for every site without intake:

- `<title>`, `meta description`, OpenGraph + Twitter card per page
- `sitemap.xml` via `@astrojs/sitemap`
- `robots.txt` (allow all, points to sitemap)
- `llms.txt` describing the site for AI crawlers
- `humans.txt`, `/.well-known/security.txt`
- JSON-LD `Organization` always; `LocalBusiness` when NAP supplied
- Favicon set, `apple-touch-icon`, `manifest.json`
- Correct `lang` attribute on `<html>` per site language
- Security headers via nginx (see template conventions)

Optional, agent decides per site:

- JSON-LD `Service`, `FAQPage`, `BreadcrumbList`, `Product`
- Multi-language (`hreflang`)
- Plausible analytics snippet (off by default, ready to enable)

## Orchestrator files

### `CLAUDE.md`

Auto-loaded by Claude Code from the sandbox. Concise list of high-level
conventions:

- Org is `optidigi`. New repos: `site-<slug>`, public, ghcr.io for images.
- Output sites are static Astro + Tailwind, served by nginx in container.
- Three subagents available: `copywriter`, `auditor`, `reviewer`. Use them
  per the runbook in `prompt.md`.
- Never modify `sitegen-template/` or `sitegen-themes/` during a run. Only
  write into `site-<slug>/`.
- Never push to `main` of `site-<slug>` until the user has approved the
  preview.
- Read `preflight.md` first. Then await user confirmation. Then read
  `prompt.md`.

### `preflight.md`

Background context the agent reads on demand at the start of a run. After
reading, the agent summarizes back what it understood and the user confirms
before the agent reads `prompt.md`.

Sections:

1. Purpose & business model
2. Tech stack & conventions (full deploy chain)
3. Standards (SEO baseline, mobile-first, WCAG 2.2 AA floor, Lighthouse
   budget)
4. Tool inventory (`gh`, `pnpm`, `node`, `docker`, `sharp`, `lighthouse-ci`,
   `@axe-core/cli`, MCPs available, subagent types available)
5. Repo locations & permissions (sibling dirs, naming, ghcr.io path, VPS
   deploy is server-side and outside agent scope)
6. Anti-patterns (no inventing themes, no push without approval, no
   stripping the SEO baseline, no large copy without dispatching
   `copywriter`)

### `prompt.md` — the runbook

Sequential phases with explicit gates.

```
PHASE 1: INTAKE
  Walk the intake checklist (Identity / Scope / Content / SEO / Brand).
  Accept "n/a" or "skip" per field. Confirm summary back to user.
  GATE: user approves intake summary.

PHASE 2: SCAFFOLD
  cp -r sitegen-template/. site-<slug>/   (excluding .git, README.md)
  Init fresh git repo in site-<slug>/.
  Set astro.config SITE_URL to primary domain.
  Wire site.ts with NAP, socials, opening hours (or stub if n/a).

PHASE 3: THEME INTEGRATION
  If theme path given → integrate from sitegen-themes/<path>/.
  If "pick for me" → propose 2-3 themes with rationale, await user pick.
  Copy/adapt theme components into src/components/.
  Map theme colors into tailwind.config.

PHASE 4: CONTENT
  Read client materials from path provided in intake.
  For each page: dispatch `copywriter` subagent with raw text, keywords,
  tone, language, page role.
  Write returned copy into src/content/pages/<page>.md.
  Place images: optimize via sharp, drop into src/assets/, reference from
  content.

PHASE 5: SEO HYDRATION
  Per page: title, meta description, og image (generate if missing).
  Generate JSON-LD: Organization always, LocalBusiness if NAP supplied.
  Update llms.txt with site description.
  Update robots.txt with sitemap URL.

PHASE 6: BUILD & AUDIT
  pnpm install, pnpm build. Fix any build errors inline.
  pnpm dev → background. Print preview URL.
  Dispatch `auditor` subagent against preview URL.
  Address must-fix audit findings; defer nice-to-have.
  Loop until must-fix is empty (max 3 iterations, then escalate).

PHASE 7: REVIEW
  Dispatch `reviewer` subagent with site dir + brief + auditor report.
  Address blocking findings (max 2 iterations, then escalate).

PHASE 8: USER SIGN-OFF
  Hand preview URL + audit summary + reviewer summary to user.
  GATE: user clicks around in own browser, approves.

PHASE 9: PUBLISH
  gh repo create optidigi/site-<slug> --public --source=site-<slug>/
  Push main. Watch GHA via gh run watch.
  Confirm image lands on ghcr.io.
  Tell user the image path: ghcr.io/optidigi/site-<slug>:latest
  GATE: user confirms VPS pulled & site is live at primary domain.

PHASE 10: CLEANUP
  rm -rf site-<slug>/. Done.
```

### Intake checklist (Phase 1)

The agent walks the full list, accepting "n/a" per field.

**Identity:** site slug, primary domain, other domains/aliases, brand name,
site language (default `nl`).

**Scope:** one-pager or multi-page (and which pages); theme to use (path
under `sitegen-themes/`, or "pick for me" with brief); forms needed (mailto
vs Web3Forms); analytics (none vs Plausible).

**Content intake:** path to client materials (local folder, Drive link,
Dropbox, etc. — whatever the user provides), tone/voice notes, "do not
rewrite" flag.

**SEO / business data:** keywords / target search intent, NAP (name, address,
phone), opening hours, service area / geo focus, social links.

**Brand:** primary/accent color (or "pull from logo"), logo path, font
preference (or "use theme default").

### `.mcp.json`

Project-scoped, currently minimal. Documents that `context7` is available
(plugin-loaded, no config needed) for library docs lookups. May add
`playwright` later if audits are promoted to use it.

### `.claude/settings.json`

Permissions allowlist for common bash calls used by the workflow:
`pnpm *`, `gh *`, `docker *`, `git *` inside child dirs, `cp -r`,
`rm -rf site-*` (specifically not unconstrained `rm -rf`). Keeps the flow
smooth without auto-approving destructive operations.

### `.claude/commands/new-site.md`

Optional `/new-site` slash command that loads `preflight.md` and starts the
confirm-and-read-prompt sequence. Pure ergonomics.

### `README.md`

Human-facing setup: how to clone the three sibling repos, where to drop
client materials, how to start a run, how the VPS side works.

## Subagent specialists

Three specialist subagents, defined in `.claude/agents/`. The main session
acts as both orchestrator and engineer (Framing 1 — orchestrator does the
mechanical building inline; only the three specialist tasks are delegated).

### `copywriter`

- **Tools:** `Read`, `Write`, `Edit`, `WebSearch`, `WebFetch`
- **Triggered:** Phase 4, one dispatch per page
- **Input:** raw client text, page slug + role, target keywords, tone notes,
  site language, word-count guidance
- **Output:** writes `src/content/pages/<slug>.md` with frontmatter (title,
  description ≤160 chars, keywords, ogImage) and structured body (H1 + H2/H3
  appropriate to page role); image alt-text suggestions inline as HTML
  comments
- **Hard rules:**
  - If "do not rewrite" flag is set, use verbatim — only restructure for
    headings/SEO
  - Match site language; never mix
  - No invented business facts (services, prices, claims) — flag gaps back
    to main agent

### `auditor`

- **Tools:** `Bash`, `Read`
- **Triggered:** Phase 6, after `pnpm dev` is running
- **Input:** preview URL + list of page paths
- **Runs:**
  - Lighthouse CLI (mobile preset) per page
  - `@axe-core/cli` per page
  - `curl -I` security header check
  - `curl` baseline check for `robots.txt`, `sitemap.xml`, `llms.txt`,
    favicon
- **Output:** prioritized markdown report

**Audit thresholds**

| Metric                       | Must-fix (blocks sign-off) | Should-fix | Nice-to-have |
| ---------------------------- | -------------------------- | ---------- | ------------ |
| Lighthouse Performance (mob) | <75                        | 75–84      | 85–89        |
| Lighthouse Accessibility     | <85                        | 85–94      | —            |
| Lighthouse Best Practices    | <85                        | 85–89      | —            |
| Lighthouse SEO               | <95                        | —          | —            |
| axe-core                     | any critical or serious    | moderate   | minor        |
| Security headers             | any missing CSP/HSTS/XCTO  | —          | —            |
| SEO baseline files           | any missing                | —          | —            |

- **Hard rules:** never modifies the site, only reports.

### `reviewer`

- **Base:** uses the existing `code-reviewer` agent type
- **Triggered:** Phase 7, after auditor passes
- **Input:** path to `site-<slug>/`, intake brief, auditor report
- **Reviews against:**
  - Standards from `preflight.md` (SEO baseline present, mobile-first, lang
    attr correct, JSON-LD valid)
  - Brief alignment (every page from scope built, NAP matches, brand colors
    applied)
  - Cleanliness (no `TODO`/`FIXME`, no `lorem ipsum`, no template
    placeholder strings, no orphaned theme files)
  - Links resolve (internal links + obvious external links)
- **Output:** blocking issues + non-blocking suggestions
- **Hard rules:** never modifies the site, only reports.

## Error handling

| Failure                          | Behavior                                                                                                                                              |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Build fails                      | Debug inline, fix, re-run. Do NOT proceed to audit until build is clean.                                                                              |
| Auditor reports must-fix         | Fix, re-run `pnpm build`, re-dispatch auditor. Loop until must-fix is empty. Max 3 loops, then escalate to user with current state.                   |
| Reviewer reports blockers        | Address; may re-run auditor if changes touched perf/a11y. Re-dispatch reviewer. Max 2 loops, then escalate.                                           |
| User rejects preview at sign-off | Collect specific feedback, return to whichever phase the feedback applies to (usually 3 or 4). Do not redo work that wasn't questioned.               |
| `gh repo create` fails           | Report error to user (likely auth or naming collision). Do not retry blindly. Do not attempt destructive workarounds.                                 |
| GHA build fails on push          | `gh run watch`, tail logs, diagnose. Code issue → fix and push again. Infra issue (token, perms) → escalate with exact error.                         |
| User says "abort" mid-run        | Stop cleanly. Do NOT delete `site-<slug>/`. Report current phase + state.                                                                             |
| Site broken on VPS post-deploy   | Out of agent scope. Can help diagnose by running the published image locally (`docker run --rm -p 8080:80 ghcr.io/optidigi/site-<slug>:latest`).      |

## State across runs

- `site-<slug>/` persists on disk if a run aborts or fails — resume by
  telling the agent which phase to start from.
- `client-input` paths are wherever the user provides; orchestrator does not
  manage them.
- After Phase 10, `site-<slug>/` is gone — the published GitHub repo and the
  ghcr.io image are the source of truth.

## Re-engagements (client wants changes weeks later)

`git clone optidigi/site-<slug>` back into `sandbox/site-<slug>/`, edit,
push. Skip Phases 1–3 (intake / scaffold / theme), jump to whichever phase
applies to the requested change (typically Phase 4 for content updates).
This preserves the commit history of the prior build.

## Out of scope (explicitly)

- VPS-side compose files, nginx vhost / reverse-proxy config, TLS certs
- DNS management
- Hosting GHA runners, secrets management beyond default `GITHUB_TOKEN`
- Multi-environment staging (dev/stage/prod) — these are single-environment
  production sites
- CMS integration — content lives in markdown, agent writes it during the run
