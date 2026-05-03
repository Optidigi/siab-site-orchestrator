# Sitegen runbook

You have read `preflight.md` and the user has confirmed your understanding.
Follow this runbook phase-by-phase. Each **GATE** marker is a hard stop —
do not proceed past it without the action specified.

---

## Phase 1 — Intake

(Note: `gh` auth was verified in `preflight.md`'s "Environment readiness" check before you got here.)

Walk the checklist below with the user, one section at a time. Accept "n/a"
or "skip" per field. After all sections, summarize back what you captured.

### Identity
- Site **slug** (used for repo name `site-<slug>` and local dir; lowercase,
  hyphenless preferred — `amicare`, not `ami-care`)
- **Primary domain** (e.g. `amicare.nl`)
- Other domains / **aliases** (for canonical / redirect notes)
- **Brand / business name** (used in titles, JSON-LD, footer)
- **Site language** (default `nl`)

### Scope
- One-pager or multi-page? If multi: list pages with their slugs
- **Theme** to use — path under `sitegen-themes/` (e.g. `astro/cleanco`),
  or "pick for me" with a brief
- **Forms** needed? (default: mailto. Web3Forms if a working POST form is
  required; user supplies the access key)
- **Analytics**? (default: none. Plausible if explicitly requested)

### Content intake
- Path / link to **client materials** (local folder, Drive, Dropbox — wherever
  the user has them). Read from there during Phase 4.
- **Tone / voice** notes
- **"Do not rewrite"** flag — if set, copywriter uses verbatim text and only
  restructures for headings/SEO

### SEO / business data
- **Keywords / target search intent**
- **NAP** — name (legal + display), street, postal, city, country, phone, email
- **Opening hours** (per day-of-week, opens/closes; required if NAP is set and
  business is physical)
- **Service area / geo focus** (e.g. `["Amsterdam", "Utrecht"]`)
- **Social links** (facebook, instagram, linkedin, youtube, x)

### Brand
- **Primary** + **accent color** (or "pull from logo")
- **Logo file** path
- **Font** preference (or "use theme default")

**GATE:** Show the captured intake as a bulleted summary. User must approve
before you scaffold anything.

---

## Phase 2 — Scaffold

Run from the orchestrator root (the directory holding this `prompt.md` — wherever the operator cloned the orchestrator).

```bash
cp -r sitegen-template/. site-<slug>/
cd site-<slug>
rm -rf .git
git init -b main
```

Then:
- Set `SITE_URL` env reference in `astro.config.mjs` if you need a non-default
  primary domain (the file already reads from `process.env.SITE_URL`).
- Open `src/content/site.ts` and replace the placeholder with intake data:
  brand, language, primaryDomain, aliases, description, NAP (if supplied),
  hours, serviceArea, socials, nav (matches scope).
- Update `public/manifest.json` (name, short_name, theme_color).
- Update `public/.well-known/security.txt` Canonical to the primary domain.

Verify: `pnpm install && pnpm astro check` succeeds.

---

## Phase 3 — Theme integration

If the user gave a theme path:
- Read `sitegen-themes/<path>/theme.json` (if present) and the theme files.
- For Astro themes: copy/adapt components into `src/components/<theme>/`. Map
  the theme's color tokens into `src/styles/global.css` `@theme` block and
  `tailwind.config.ts` `brand`/`accent` colors.
- For plain HTML/CSS/JS: adapt into Astro components — extract recurring
  blocks (header, footer, hero, feature row) into `src/components/`. Don't
  paste raw HTML into pages; componentize.

If the user said "pick for me":
- Read each `theme.json` in `sitegen-themes/{astro,plain}/` and propose 2–3
  themes that match the brief, with one-line rationale each.
- Wait for user pick, then proceed as above.

Verify: `pnpm dev`, eyeball at `http://localhost:4321`. Header/footer/nav
match theme. Tailwind colors hot-reload. Build `pnpm build` succeeds.

---

## Phase 4 — Content

Read raw client materials from the path captured in intake.

For each page in scope:
1. Group the relevant raw text + any per-page constraints.
2. Dispatch the `copywriter` subagent with:
   - raw text (verbatim)
   - page slug + role
   - target keywords
   - tone notes (and "do not rewrite" flag if set)
   - site language
   - word-count guidance per section (rough is fine)
3. Subagent writes `src/content/pages/<slug>.md`.
4. If subagent flags missing facts (services, prices, claims), surface those
   to the user before moving on.

Images:
- Optimize raw images via `sharp` (resize to ≤2400px wide, convert to WebP if
  source is heavier). Save to `src/assets/<page>/`.
- Reference from markdown via Astro's `<Image>` (or `![]()` if rendered as
  static — `<Image>` preferred for non-static layout components).

Make sure each page in scope has a corresponding `.astro` route under
`src/pages/` that pulls from `src/content/pages/<slug>.md`.

---

## Phase 5 — SEO hydration

For each page:
- Confirm frontmatter `title` (≤70 chars) and `description` (≤160 chars).
- If `ogImage` is missing for a page that needs a custom one, generate via
  `sharp` from a brand template (or fall back to `/og-default.png`).
- Update `public/og-default.png` to a brand-colored 1200×630 image with the
  brand name (use `sharp` or imagemagick).

Site-wide:
- `public/llms.txt`: replace placeholder with a 3–5 paragraph site
  description suitable for AI crawlers (what the business does, who it
  serves, where, contact).
- `robots.txt` is dynamic via `src/pages/robots.txt.ts` — already points to
  `${SITE_URL}/sitemap-index.xml`; nothing to change unless you need a
  per-section disallow.

JSON-LD:
- `JsonLdOrganization.astro` reads from `site.ts` automatically.
- `JsonLdLocalBusiness.astro` only renders when `site.nap` is set — confirm
  it's set if the business is physical.
- Add `JsonLdBreadcrumbList`, `JsonLdFAQPage`, etc. inline only if the page
  warrants it (FAQs only on FAQ-style pages).

---

## Phase 6 — Build & audit

```bash
pnpm build        # fix any errors before continuing
pnpm dev          # in background — capture the URL it prints
```

Dispatch the `auditor` subagent with the preview URL and the list of pages.
Auditor returns a markdown report with **must-fix / should-fix / nice-to-have**
sections.

- Address all **must-fix** items.
- Re-run `pnpm build` after each round of fixes.
- Re-dispatch auditor.
- Loop until must-fix list is empty. **Max 3 loops** — if not clean after 3,
  stop and escalate to the user with current state and what's stuck.

Defer should-fix unless cheap; defer nice-to-have unless trivial.

---

## Phase 7 — Review

Dispatch the `reviewer` subagent (uses `code-reviewer` type) with:
- path to `site-<slug>/`
- the captured intake brief
- the latest auditor report

Reviewer returns blocking + non-blocking findings.

- Address blocking findings.
- If fixes touched perf or a11y, re-run auditor briefly.
- Re-dispatch reviewer. **Max 2 loops.**

---

## Phase 8 — User sign-off (GATE)

Hand the user:
- The preview URL (`http://localhost:4321`).
- A short summary of the auditor's final report (Lighthouse scores,
  outstanding should-fix items if any).
- A short summary of the reviewer's outcome.
- The list of pages built.

Ask the user to click around in their own browser. **Wait for explicit
approval** before Phase 9. If user requests changes, return to whichever
phase the change belongs in (usually 3 or 4) and re-run downstream phases.

Stop the dev server before moving on.

---

## Phase 9 — Publish

From the orchestrator root:

```bash
cd site-<slug>
git add -A
git commit -m "feat: initial site"
gh repo create optidigi/site-<slug> --public --source=. --remote=origin --push
gh run watch --exit-status
```

After the workflow finishes:
- Confirm the image landed: `gh api /orgs/optidigi/packages/container/site-<slug>/versions | head -50`
  (or check https://github.com/orgs/optidigi/packages).
- Tell the user the image path: `ghcr.io/optidigi/site-<slug>:latest`.
- **GATE:** wait for the user to confirm the VPS pulled the image and the
  primary domain serves the site.

If the user reports a problem deploying, you can help diagnose by running the
published image locally:
```bash
docker pull ghcr.io/optidigi/site-<slug>:latest
docker run --rm -p 8080:80 ghcr.io/optidigi/site-<slug>:latest
# then curl localhost:8080
```

---

## Phase 10 — Cleanup

From the orchestrator root:

```bash
rm -rf site-<slug>/
```

Confirm to the user the site is shipped, the image is at
`ghcr.io/optidigi/site-<slug>:latest`, the source repo is at
`https://github.com/optidigi/site-<slug>`, and your local working dir is
clean.

Done.
