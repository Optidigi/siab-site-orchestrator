# Sitegen Orchestrator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the four-repo workflow defined in `docs/superpowers/specs/2026-05-02-sitegen-orchestrator-design.md` so we can spin up Astro landing-page sites end-to-end, with each site landing as `optidigi/site-<slug>` and publishing a Docker image to `ghcr.io/optidigi/site-<slug>`.

**Architecture:** Sandbox dir is a git repo (`optidigi/sitegen-orchestrator`) holding workflow files. Two sibling clones (`sitegen-template` for the Astro boilerplate, `sitegen-themes` for theme building blocks) are gitignored from the orchestrator. Per-engagement, the agent copies the template into `sandbox/site-<slug>/`, integrates a theme, fills content via three subagent specialists (`copywriter`, `auditor`, `reviewer`), pushes to a fresh `optidigi/site-<slug>` repo, and watches GitHub Actions publish a static-build → nginx Docker image to ghcr.io. VPS-side compose pulls and runs.

**Tech Stack:** Astro 5.x (static output), Tailwind CSS 4 (`@tailwindcss/vite`), `@astrojs/sitemap`, `astro-seo`, pnpm, Docker (multi-stage `node:lts-alpine` → `nginx:alpine`), GitHub Actions (`docker/build-push-action` to ghcr.io with `:latest` + `:sha-<short>` via `docker/metadata-action`), Web3Forms for optional contact forms, Lighthouse CLI + `@axe-core/cli` for audits.

---

## File Layout (created by this plan)

**Orchestrator repo (`sandbox/`)**
```
.gitignore
README.md
CLAUDE.md
preflight.md
prompt.md
.mcp.json
.claude/settings.json
.claude/agents/copywriter.md
.claude/agents/auditor.md
.claude/agents/reviewer.md
.claude/commands/new-site.md
docs/superpowers/specs/2026-05-02-sitegen-orchestrator-design.md  (already on disk)
docs/superpowers/plans/2026-05-02-sitegen-orchestrator.md          (this file)
```

**Template repo (`sandbox/sitegen-template/`)**
```
package.json
pnpm-lock.yaml          (generated)
tsconfig.json
astro.config.mjs
tailwind.config.ts
.gitignore
.dockerignore
.env.example
Dockerfile
nginx.conf
.github/workflows/publish.yml
README.md
src/styles/global.css
src/layouts/BaseLayout.astro
src/components/seo/Seo.astro
src/components/seo/JsonLdOrganization.astro
src/components/seo/JsonLdLocalBusiness.astro
src/components/forms/ContactForm.astro
src/content/config.ts
src/content/site.ts
src/content/pages/index.md
src/pages/index.astro
src/pages/404.astro
src/pages/robots.txt.ts
public/llms.txt
public/humans.txt
public/.well-known/security.txt
public/manifest.json
public/favicon.svg              (placeholder)
public/apple-touch-icon.png     (placeholder, 180x180)
public/og-default.png           (placeholder, 1200x630)
```

**Themes repo (`sandbox/sitegen-themes/`)**
```
.gitignore
README.md
astro/.gitkeep
plain/.gitkeep
```

---

## Task 1: Initialize orchestrator repo skeleton

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/.gitignore`
- Create: `/home/shimmy/Desktop/env/sandbox/README.md`
- Init git: `/home/shimmy/Desktop/env/sandbox/`

**Why:** Get the sandbox dir under version control so the spec + this plan land as the first commit, and the workflow files we're about to create have a home.

- [ ] **Step 1: Initialize git repo in sandbox**

Run: `cd /home/shimmy/Desktop/env/sandbox && git init -b main`
Expected: `Initialized empty Git repository in /home/shimmy/Desktop/env/sandbox/.git/`

- [ ] **Step 2: Create `.gitignore`**

Create `/home/shimmy/Desktop/env/sandbox/.gitignore`:

```gitignore
# Sibling repos cloned alongside this orchestrator (have their own git)
/sitegen-template/
/sitegen-themes/
/site-*/

# Per-client raw materials, wherever the user drops them locally
/client-input/
/_clients/

# Local Claude state
.claude/projects/
.claude/shell-snapshots/
.claude/todos/
.claude/statsig/
.claude/ide/

# OS / editor noise
.DS_Store
Thumbs.db
*.swp
.idea/
.vscode/
```

- [ ] **Step 3: Create `README.md`** (human-facing setup; full content lands in Task 11)

Create `/home/shimmy/Desktop/env/sandbox/README.md`:

```markdown
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
```

- [ ] **Step 4: Verify spec & plan are present, then commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
ls docs/superpowers/specs/2026-05-02-sitegen-orchestrator-design.md
ls docs/superpowers/plans/2026-05-02-sitegen-orchestrator.md
git add .gitignore README.md docs/
git status
```
Expected: both spec and plan listed under "new file"; `.gitignore`, `README.md`, `docs/` staged.

- [ ] **Step 5: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
git commit -m "chore: initialize sitegen-orchestrator with spec and plan"
```
Expected: commit succeeds with the 4 files (`.gitignore`, `README.md`, spec, plan).

---

## Task 2: Initialize sitegen-themes (smallest, gets it out of the way)

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-themes/.gitignore`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-themes/README.md`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-themes/astro/.gitkeep`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-themes/plain/.gitkeep`

**Why:** The orchestrator's runbook references `sitegen-themes/{astro,plain}/` paths. We need the directory structure to exist so paths in `preflight.md` and `prompt.md` aren't speculative.

- [ ] **Step 1: Create directory and init git**

Run:
```bash
mkdir -p /home/shimmy/Desktop/env/sandbox/sitegen-themes/{astro,plain}
cd /home/shimmy/Desktop/env/sandbox/sitegen-themes
git init -b main
```
Expected: `Initialized empty Git repository`.

- [ ] **Step 2: Create `.gitignore`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-themes/.gitignore`:

```gitignore
# Theme work in progress
node_modules/
.DS_Store
*.swp

# If a theme has its own build artifacts, theme authors should add per-theme .gitignore
```

- [ ] **Step 3: Create `README.md`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-themes/README.md`:

```markdown
# sitegen-themes

Curated building blocks for the sitegen workflow. Used by `optidigi/sitegen-orchestrator` to source theme components when generating sites.

## Layout

```
astro/<theme-slug>/   # Astro themes — preferred. Use components/islands.
plain/<theme-slug>/   # Plain HTML/CSS/JS templates — adapt into Astro components per-engagement.
```

## Adding a theme

1. Drop the theme into the appropriate subdir.
2. Add a `theme.json` at its root with at minimum:
   - `name`, `summary`, `tags[]` (e.g. `["dental", "minimal", "single-page"]`)
   - `license`, `source` (URL)
   - `palette`: array of `{name, hex}` for primary/accent colors
3. Commit and push.

The orchestrator agent reads `theme.json` files to suggest themes when the client says "pick for me".
```

- [ ] **Step 4: Add `.gitkeep` placeholders**

Run:
```bash
touch /home/shimmy/Desktop/env/sandbox/sitegen-themes/astro/.gitkeep
touch /home/shimmy/Desktop/env/sandbox/sitegen-themes/plain/.gitkeep
```

- [ ] **Step 5: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-themes
git add .
git commit -m "chore: initialize sitegen-themes with astro/ and plain/ subdirs"
```
Expected: commit succeeds with 4 files.

---

## Task 3: Initialize sitegen-template — bare Astro scaffold

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/` (whole tree, scaffolded by Astro CLI then customized)

**Why:** Get a working `pnpm create astro@latest` shell so subsequent tasks layer onto a known-good baseline.

- [ ] **Step 1: Verify pnpm + node available**

Run: `node --version && pnpm --version`
Expected: Node 20.x or newer, pnpm 9.x or newer. If pnpm is missing: `corepack enable && corepack prepare pnpm@latest --activate`.

- [ ] **Step 2: Scaffold a minimal Astro project (non-interactive)**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
pnpm create astro@latest sitegen-template -- --template minimal --typescript strict --install --no-git --skip-houston
```
Expected: a `sitegen-template/` dir with a working Astro minimal template, `node_modules/` installed.

- [ ] **Step 3: Init git in the scaffold**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git init -b main
```
Expected: `Initialized empty Git repository`.

- [ ] **Step 4: Verify it runs**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
pnpm build
```
Expected: build succeeds, `dist/` populated with at least an `index.html`.

- [ ] **Step 5: Commit baseline**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git add -A
git commit -m "chore: scaffold minimal Astro template"
```
Expected: commit succeeds with the scaffold.

---

## Task 4: Add Tailwind CSS 4 + integrations + deps

**Files:**
- Modify: `/home/shimmy/Desktop/env/sandbox/sitegen-template/package.json`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/tailwind.config.ts`
- Modify: `/home/shimmy/Desktop/env/sandbox/sitegen-template/astro.config.mjs`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/styles/global.css`

**Why:** Tailwind 4 + sitemap + astro-seo are the foundational integrations the SEO components and theme work in later tasks depend on.

- [ ] **Step 1: Install dependencies**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
pnpm add @astrojs/sitemap astro-seo
pnpm add -D tailwindcss @tailwindcss/vite @tailwindcss/typography @astrojs/check typescript
```
Expected: deps land in `package.json`, lockfile updated.

- [ ] **Step 2: Replace `astro.config.mjs`**

Replace `/home/shimmy/Desktop/env/sandbox/sitegen-template/astro.config.mjs`:

```javascript
import { defineConfig } from 'astro/config';
import sitemap from '@astrojs/sitemap';
import tailwindcss from '@tailwindcss/vite';

const SITE_URL = process.env.SITE_URL ?? 'https://example.com';

export default defineConfig({
  site: SITE_URL,
  output: 'static',
  integrations: [sitemap()],
  vite: {
    plugins: [tailwindcss()],
  },
  build: {
    inlineStylesheets: 'auto',
  },
});
```

- [ ] **Step 3: Create `tailwind.config.ts`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/tailwind.config.ts`:

```typescript
import type { Config } from 'tailwindcss';

export default {
  content: ['./src/**/*.{astro,html,js,jsx,md,mdx,svelte,ts,tsx,vue}'],
  theme: {
    extend: {
      colors: {
        // theme integration overrides these per site
        brand: {
          DEFAULT: '#0ea5e9',
          fg: '#ffffff',
        },
        accent: {
          DEFAULT: '#f59e0b',
          fg: '#000000',
        },
      },
    },
  },
  plugins: [],
} satisfies Config;
```

- [ ] **Step 4: Create `src/styles/global.css`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/styles/global.css`:

```css
@import 'tailwindcss';
@plugin '@tailwindcss/typography';

@theme {
  --color-brand: #0ea5e9;
  --color-brand-fg: #ffffff;
  --color-accent: #f59e0b;
  --color-accent-fg: #000000;
}

html {
  -webkit-text-size-adjust: 100%;
  scroll-behavior: smooth;
}

body {
  font-family: system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
  text-rendering: optimizeLegibility;
}
```

- [ ] **Step 5: Verify build still passes**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
pnpm build
```
Expected: build succeeds, no Tailwind errors.

- [ ] **Step 6: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git add -A
git commit -m "feat: add Tailwind 4, astro-seo, sitemap integrations"
```

---

## Task 5: Content collections schema + site-wide data

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/content/config.ts`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/content/site.ts`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/content/pages/index.md`

**Why:** Defines the typed shape for per-page markdown and the central `site` object that drives JSON-LD, footer, contact info. Everything that follows reads from these.

- [ ] **Step 1: Create `src/content/config.ts`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/content/config.ts`:

```typescript
import { defineCollection, z } from 'astro:content';

const pages = defineCollection({
  type: 'content',
  schema: z.object({
    title: z.string().min(1).max(70),
    description: z.string().min(1).max(160),
    keywords: z.array(z.string()).default([]),
    ogImage: z.string().optional(),
    role: z.enum(['home', 'about', 'services', 'contact', 'page']).default('page'),
    draft: z.boolean().default(false),
    order: z.number().default(0),
  }),
});

export const collections = { pages };
```

- [ ] **Step 2: Create `src/content/site.ts`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/content/site.ts`:

```typescript
export type NAP = {
  legalName: string;
  displayName: string;
  street: string;
  postalCode: string;
  city: string;
  country: string;          // ISO-3166-1 alpha-2
  phone: string;            // E.164 preferred
  email: string;
};

export type OpeningHours = {
  dayOfWeek: 'Mo' | 'Tu' | 'We' | 'Th' | 'Fr' | 'Sa' | 'Su';
  opens: string;            // 'HH:MM'
  closes: string;
};

export type SiteConfig = {
  brand: string;
  language: string;         // 'nl' | 'en' | etc.
  primaryDomain: string;    // 'amicare.nl'
  aliases: string[];        // ['ami-care.nl', 'a-m-i-care.com']
  description: string;
  nap?: NAP;
  hours?: OpeningHours[];
  serviceArea?: string[];   // ['Amsterdam', 'Utrecht']
  socials: {
    facebook?: string;
    instagram?: string;
    linkedin?: string;
    youtube?: string;
    x?: string;
  };
  nav: { label: string; href: string }[];
};

export const site: SiteConfig = {
  brand: 'Example Brand',
  language: 'nl',
  primaryDomain: 'example.com',
  aliases: [],
  description: 'Replace this description per site during Phase 1 intake.',
  socials: {},
  nav: [
    { label: 'Home', href: '/' },
  ],
};
```

- [ ] **Step 3: Create `src/content/pages/index.md`** (placeholder homepage)

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/content/pages/index.md`:

```markdown
---
title: Welcome
description: Replace this with the per-site homepage description during Phase 4 of the runbook.
keywords: []
role: home
order: 0
---

# Welcome

This is the placeholder homepage shipped with `sitegen-template`. The orchestrator's `copywriter` subagent overwrites this file during Phase 4 (Content) of the runbook.

If you're seeing this on a deployed site, content generation didn't run — check the agent logs.
```

- [ ] **Step 4: Verify build picks up content**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
pnpm build
```
Expected: build succeeds, no zod schema errors. (Build won't fail just because nothing renders pages from the collection yet — that comes in Task 7.)

- [ ] **Step 5: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git add -A
git commit -m "feat: add typed content collections + site config"
```

---

## Task 6: SEO components — central Seo + JSON-LD generators

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/components/seo/Seo.astro`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/components/seo/JsonLdOrganization.astro`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/components/seo/JsonLdLocalBusiness.astro`

**Why:** This is the SEO baseline guarantee — every page goes through `Seo.astro`, every site emits `Organization` JSON-LD, sites with NAP also emit `LocalBusiness`.

- [ ] **Step 1: Create `Seo.astro`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/components/seo/Seo.astro`:

```astro
---
import { SEO } from 'astro-seo';
import { site } from '../../content/site';

interface Props {
  title: string;
  description: string;
  ogImage?: string;          // relative path or absolute URL
  noindex?: boolean;
  canonical?: string;
}

const { title, description, ogImage, noindex = false, canonical } = Astro.props;

const ogImageUrl = new URL(ogImage ?? '/og-default.png', Astro.site).toString();
const canonicalUrl = canonical ?? Astro.url.toString();
const fullTitle = title === site.brand ? title : `${title} | ${site.brand}`;
---

<SEO
  title={fullTitle}
  description={description}
  canonical={canonicalUrl}
  noindex={noindex}
  openGraph={{
    basic: {
      title: fullTitle,
      type: 'website',
      image: ogImageUrl,
      url: canonicalUrl,
    },
    optional: {
      siteName: site.brand,
      description,
      locale: site.language,
    },
  }}
  twitter={{
    card: 'summary_large_image',
    title: fullTitle,
    description,
    image: ogImageUrl,
  }}
/>
```

- [ ] **Step 2: Create `JsonLdOrganization.astro`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/components/seo/JsonLdOrganization.astro`:

```astro
---
import { site } from '../../content/site';

const data = {
  '@context': 'https://schema.org',
  '@type': 'Organization',
  name: site.brand,
  url: `https://${site.primaryDomain}`,
  description: site.description,
  sameAs: Object.values(site.socials).filter(Boolean),
};
---

<script type="application/ld+json" set:html={JSON.stringify(data)} />
```

- [ ] **Step 3: Create `JsonLdLocalBusiness.astro`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/components/seo/JsonLdLocalBusiness.astro`:

```astro
---
import { site } from '../../content/site';

if (!site.nap) {
  // No NAP supplied → render nothing.
}

const dayMap: Record<string, string> = {
  Mo: 'Monday', Tu: 'Tuesday', We: 'Wednesday', Th: 'Thursday',
  Fr: 'Friday', Sa: 'Saturday', Su: 'Sunday',
};

const data = site.nap
  ? {
      '@context': 'https://schema.org',
      '@type': 'LocalBusiness',
      name: site.nap.displayName,
      legalName: site.nap.legalName,
      url: `https://${site.primaryDomain}`,
      telephone: site.nap.phone,
      email: site.nap.email,
      address: {
        '@type': 'PostalAddress',
        streetAddress: site.nap.street,
        postalCode: site.nap.postalCode,
        addressLocality: site.nap.city,
        addressCountry: site.nap.country,
      },
      areaServed: site.serviceArea,
      openingHoursSpecification: site.hours?.map((h) => ({
        '@type': 'OpeningHoursSpecification',
        dayOfWeek: dayMap[h.dayOfWeek],
        opens: h.opens,
        closes: h.closes,
      })),
    }
  : null;
---

{data && <script type="application/ld+json" set:html={JSON.stringify(data)} />}
```

- [ ] **Step 4: Verify the three files compile (build with current pages — won't render yet, just no syntax errors)**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
pnpm astro check
```
Expected: 0 errors. Warnings about unused exports are fine.

- [ ] **Step 5: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git add -A
git commit -m "feat: add Seo + JSON-LD components (Organization + LocalBusiness)"
```

---

## Task 7: BaseLayout + index page + 404 + dynamic robots.txt

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/layouts/BaseLayout.astro`
- Modify: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/pages/index.astro` (replace minimal scaffold)
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/pages/404.astro`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/pages/robots.txt.ts`

**Why:** Wire SEO components into a layout, make the homepage actually render content from the collection, and emit a robots.txt that points to the sitemap.

- [ ] **Step 1: Create `BaseLayout.astro`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/layouts/BaseLayout.astro`:

```astro
---
import '../styles/global.css';
import Seo from '../components/seo/Seo.astro';
import JsonLdOrganization from '../components/seo/JsonLdOrganization.astro';
import JsonLdLocalBusiness from '../components/seo/JsonLdLocalBusiness.astro';
import { site } from '../content/site';

interface Props {
  title: string;
  description: string;
  ogImage?: string;
  noindex?: boolean;
}

const { title, description, ogImage, noindex } = Astro.props;
---

<!doctype html>
<html lang={site.language}>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
    <link rel="apple-touch-icon" href="/apple-touch-icon.png" />
    <link rel="manifest" href="/manifest.json" />
    <meta name="generator" content={Astro.generator} />
    <Seo title={title} description={description} ogImage={ogImage} noindex={noindex} />
    <JsonLdOrganization />
    <JsonLdLocalBusiness />
  </head>
  <body class="bg-white text-slate-900 antialiased">
    <slot />
  </body>
</html>
```

- [ ] **Step 2: Replace `src/pages/index.astro`**

Replace `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/pages/index.astro`:

```astro
---
import { getEntry } from 'astro:content';
import BaseLayout from '../layouts/BaseLayout.astro';

const home = await getEntry('pages', 'index');
const { Content } = await home.render();
---

<BaseLayout
  title={home.data.title}
  description={home.data.description}
  ogImage={home.data.ogImage}
>
  <main class="prose mx-auto max-w-3xl px-4 py-16">
    <Content />
  </main>
</BaseLayout>
```

- [ ] **Step 3: Create `src/pages/404.astro`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/pages/404.astro`:

```astro
---
import BaseLayout from '../layouts/BaseLayout.astro';
---

<BaseLayout
  title="Page not found"
  description="The page you were looking for doesn't exist."
  noindex={true}
>
  <main class="prose mx-auto max-w-3xl px-4 py-24 text-center">
    <h1>404 — Page not found</h1>
    <p>The page you were looking for doesn't exist.</p>
    <p><a href="/">Back to home</a></p>
  </main>
</BaseLayout>
```

- [ ] **Step 4: Create `src/pages/robots.txt.ts`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/pages/robots.txt.ts`:

```typescript
import type { APIRoute } from 'astro';

export const GET: APIRoute = ({ site }) => {
  const sitemapUrl = new URL('sitemap-index.xml', site).toString();
  const body = `User-agent: *\nAllow: /\n\nSitemap: ${sitemapUrl}\n`;
  return new Response(body, {
    status: 200,
    headers: { 'Content-Type': 'text/plain; charset=utf-8' },
  });
};
```

- [ ] **Step 5: Build and inspect output**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
SITE_URL=https://example.com pnpm build
```
Expected: build succeeds. `dist/` contains `index.html`, `404.html`, `robots.txt`, `sitemap-index.xml`, `sitemap-0.xml`.

- [ ] **Step 6: Spot-check generated HTML**

Run:
```bash
grep -E '(application/ld\+json|<title>|og:image|lang=)' /home/shimmy/Desktop/env/sandbox/sitegen-template/dist/index.html | head -20
cat /home/shimmy/Desktop/env/sandbox/sitegen-template/dist/robots.txt
```
Expected: HTML contains `<html lang="nl">`, `<title>Welcome | Example Brand</title>`, an `og:image` meta, and at least one `application/ld+json` script. robots.txt contains `Sitemap: https://example.com/sitemap-index.xml`.

- [ ] **Step 7: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git add -A
git commit -m "feat: add BaseLayout, index, 404, robots.txt"
```

---

## Task 8: Public assets — llms.txt, humans.txt, security.txt, manifest, placeholders

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/llms.txt`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/humans.txt`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/.well-known/security.txt`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/manifest.json`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/favicon.svg`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/apple-touch-icon.png` (placeholder)
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/og-default.png` (placeholder)

**Why:** Static files the SEO baseline relies on. Each gets per-site customization in Phase 5 of the runbook; the template ships sane defaults so a freshly-scaffolded site already passes the Lighthouse SEO audit.

- [ ] **Step 1: Create `public/llms.txt`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/llms.txt`:

```
# Site description for AI crawlers
# Replace this content during Phase 5 of the sitegen runbook.

This is a placeholder. Per-site llms.txt is generated based on the intake brief.
```

- [ ] **Step 2: Create `public/humans.txt`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/humans.txt`:

```
/* TEAM */
  Site by: optidigi
  Site: https://optidigi.nl

/* TECHNOLOGY */
  Astro, Tailwind CSS, nginx, Docker, GitHub Actions
```

- [ ] **Step 3: Create `public/.well-known/security.txt`**

Run: `mkdir -p /home/shimmy/Desktop/env/sandbox/sitegen-template/public/.well-known`

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/.well-known/security.txt`:

```
Contact: mailto:security@optidigi.nl
Preferred-Languages: en, nl
Canonical: https://example.com/.well-known/security.txt
Expires: 2099-12-31T23:59:59.000Z
```

(Per-site Phase 5 fills in the correct Canonical URL.)

- [ ] **Step 4: Create `public/manifest.json`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/manifest.json`:

```json
{
  "name": "Example Brand",
  "short_name": "Example",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0ea5e9",
  "icons": [
    { "src": "/apple-touch-icon.png", "sizes": "180x180", "type": "image/png" }
  ]
}
```

- [ ] **Step 5: Create `public/favicon.svg`** (simple geometric placeholder)

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/public/favicon.svg`:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64">
  <rect width="64" height="64" rx="12" fill="#0ea5e9"/>
  <text x="32" y="42" text-anchor="middle" font-family="system-ui, sans-serif"
        font-size="32" font-weight="700" fill="#ffffff">S</text>
</svg>
```

- [ ] **Step 6: Generate placeholder PNG assets via ImageMagick or a tiny script**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
# Try ImageMagick first
if command -v magick >/dev/null 2>&1; then
  magick -size 180x180 xc:'#0ea5e9' -gravity center -fill white -font DejaVu-Sans-Bold -pointsize 96 -annotate 0 'S' public/apple-touch-icon.png
  magick -size 1200x630 xc:'#0ea5e9' -gravity center -fill white -font DejaVu-Sans-Bold -pointsize 96 -annotate 0 'Example Brand' public/og-default.png
elif command -v convert >/dev/null 2>&1; then
  convert -size 180x180 xc:'#0ea5e9' -gravity center -fill white -font DejaVu-Sans-Bold -pointsize 96 -annotate 0 'S' public/apple-touch-icon.png
  convert -size 1200x630 xc:'#0ea5e9' -gravity center -fill white -font DejaVu-Sans-Bold -pointsize 96 -annotate 0 'Example Brand' public/og-default.png
else
  # Fallback: tiny solid PNGs via Node + sharp (already a transitive Astro dep)
  node -e "
    const sharp = require('sharp');
    Promise.all([
      sharp({create:{width:180,height:180,channels:3,background:'#0ea5e9'}}).png().toFile('public/apple-touch-icon.png'),
      sharp({create:{width:1200,height:630,channels:3,background:'#0ea5e9'}}).png().toFile('public/og-default.png'),
    ]).then(()=>console.log('placeholders written'))
  "
fi
ls -la public/apple-touch-icon.png public/og-default.png
```
Expected: both PNG files exist. Sizes are correct (180x180 and 1200x630).

- [ ] **Step 7: Build and verify all assets ship into dist**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
SITE_URL=https://example.com pnpm build
ls dist/llms.txt dist/humans.txt dist/.well-known/security.txt dist/manifest.json dist/favicon.svg dist/apple-touch-icon.png dist/og-default.png
```
Expected: all 7 files present in `dist/`.

- [ ] **Step 8: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git add -A
git commit -m "feat: add public assets (llms.txt, humans.txt, security.txt, manifest, favicon, og-default)"
```

---

## Task 9: ContactForm component (mailto default, Web3Forms when key set)

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/components/forms/ContactForm.astro`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/.env.example`

**Why:** Optional contact form sites can drop in. Default is a mailto fallback — no third-party dependency unless explicitly enabled per-site.

- [ ] **Step 1: Create `.env.example`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/.env.example`:

```bash
# Set to your site's canonical URL (https://yourdomain.com). Used for canonical/OG.
SITE_URL=https://example.com

# Optional: enable the Web3Forms-backed contact form. Get a free key at https://web3forms.com.
# When unset, ContactForm.astro renders a mailto: link instead.
PUBLIC_WEB3FORMS_KEY=

# Optional: contact email used by mailto fallback. Defaults to site.nap.email if set.
PUBLIC_CONTACT_EMAIL=
```

- [ ] **Step 2: Create `ContactForm.astro`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/src/components/forms/ContactForm.astro`:

```astro
---
import { site } from '../../content/site';

interface Props {
  heading?: string;
  submitLabel?: string;
}

const { heading = 'Contact us', submitLabel = 'Send' } = Astro.props;

const web3FormsKey = import.meta.env.PUBLIC_WEB3FORMS_KEY;
const fallbackEmail = import.meta.env.PUBLIC_CONTACT_EMAIL ?? site.nap?.email;

const useWeb3Forms = Boolean(web3FormsKey);
---

<section class="mx-auto max-w-xl">
  <h2 class="text-2xl font-semibold mb-6">{heading}</h2>

  {useWeb3Forms ? (
    <form action="https://api.web3forms.com/submit" method="POST" class="space-y-4">
      <input type="hidden" name="access_key" value={web3FormsKey} />
      <input type="hidden" name="from_name" value={site.brand} />
      <input type="hidden" name="subject" value={`New message via ${site.primaryDomain}`} />

      <label class="block">
        <span class="block text-sm font-medium">Name</span>
        <input required name="name" type="text" class="mt-1 w-full rounded border-slate-300 px-3 py-2" />
      </label>

      <label class="block">
        <span class="block text-sm font-medium">Email</span>
        <input required name="email" type="email" class="mt-1 w-full rounded border-slate-300 px-3 py-2" />
      </label>

      <label class="block">
        <span class="block text-sm font-medium">Message</span>
        <textarea required name="message" rows="5" class="mt-1 w-full rounded border-slate-300 px-3 py-2"></textarea>
      </label>

      <input type="checkbox" name="botcheck" class="hidden" tabindex="-1" autocomplete="off" />

      <button type="submit" class="rounded bg-brand px-5 py-2 text-brand-fg font-medium hover:opacity-90">
        {submitLabel}
      </button>
    </form>
  ) : fallbackEmail ? (
    <p>
      Email us directly:
      <a href={`mailto:${fallbackEmail}`} class="text-brand underline">{fallbackEmail}</a>
    </p>
  ) : (
    <p class="text-sm text-slate-500">Contact form not configured.</p>
  )}
</section>
```

- [ ] **Step 3: Verify build passes (component is unused at template level — that's fine, themes/pages opt in)**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
pnpm astro check
```
Expected: 0 errors.

- [ ] **Step 4: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git add -A
git commit -m "feat: add ContactForm component (Web3Forms with mailto fallback)"
```

---

## Task 10: Dockerfile + .dockerignore + nginx.conf

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/Dockerfile`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/.dockerignore`
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/nginx.conf`

**Why:** Per spec: multi-stage build → nginx:alpine serving `dist/`, with security headers, gzip, sane caching.

- [ ] **Step 1: Create `.dockerignore`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/.dockerignore`:

```
node_modules
dist
.astro
.git
.github
.env
.env.*
!.env.example
*.md
.DS_Store
.vscode
.idea
```

- [ ] **Step 2: Create `nginx.conf`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/nginx.conf`:

```nginx
server {
    listen 80 default_server;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # Security headers
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
    add_header Content-Security-Policy "default-src 'self'; img-src 'self' data: https:; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; font-src 'self' data:; connect-src 'self' https://api.web3forms.com" always;
    # HSTS only meaningful behind TLS — your reverse proxy will add it; uncomment if serving TLS direct.
    # add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # gzip
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/json application/xml application/xml+rss application/atom+xml image/svg+xml;

    # Hashed assets — long cache
    location ~* \.(?:js|css|woff2?|ttf|eot|otf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    # Images — moderate cache, allow revalidation
    location ~* \.(?:png|jpg|jpeg|gif|webp|avif|svg|ico)$ {
        expires 30d;
        add_header Cache-Control "public";
        try_files $uri =404;
    }

    # HTML — never cache
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        try_files $uri =404;
    }

    # Plain-text infrastructure files
    location = /robots.txt    { default_type text/plain; }
    location = /sitemap.xml   { default_type application/xml; }
    location = /sitemap-index.xml { default_type application/xml; }
    location = /llms.txt      { default_type text/plain; }
    location = /humans.txt    { default_type text/plain; }
    location = /.well-known/security.txt { default_type text/plain; }

    # Default: try file → directory → 404 page
    location / {
        try_files $uri $uri/ $uri.html /404.html;
    }

    error_page 404 /404.html;
}
```

- [ ] **Step 3: Create `Dockerfile`**

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1.7
ARG NODE_VERSION=lts-alpine
ARG NGINX_VERSION=alpine

FROM node:${NODE_VERSION} AS build
WORKDIR /app

# Install deps with cache-friendly layering
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

# Build
COPY . .
ARG SITE_URL=https://example.com
ENV SITE_URL=${SITE_URL}
RUN pnpm build

FROM nginx:${NGINX_VERSION}
RUN rm -f /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

# Healthcheck via curl-less wget bundled with alpine nginx images
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://127.0.0.1/ >/dev/null || exit 1

EXPOSE 80
```

- [ ] **Step 4: Build the image locally**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
docker build --build-arg SITE_URL=https://example.com -t sitegen-template:smoke .
```
Expected: build succeeds, image tagged `sitegen-template:smoke`. Note final image size — should be <50MB.

- [ ] **Step 5: Run container and smoke-test**

Run:
```bash
docker run --rm -d -p 8080:80 --name sitegen-smoke sitegen-template:smoke
sleep 1
curl -sf -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:8080/
curl -sI http://127.0.0.1:8080/ | grep -iE '(x-content-type-options|x-frame-options|content-security-policy)'
curl -sf http://127.0.0.1:8080/robots.txt | head -3
curl -sf http://127.0.0.1:8080/llms.txt | head -2
curl -sf -o /dev/null -w '404 page: HTTP %{http_code}\n' http://127.0.0.1:8080/this-does-not-exist
docker stop sitegen-smoke
```
Expected:
- Root returns HTTP 200.
- Security headers all present.
- robots.txt and llms.txt return their content.
- Missing path returns HTTP 404 (and serves the 404.html body).

- [ ] **Step 6: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git add -A
git commit -m "feat: add Dockerfile, .dockerignore, nginx.conf with security headers and caching"
```

---

## Task 11: GitHub Actions workflow — publish to ghcr.io on push to main

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/sitegen-template/.github/workflows/publish.yml`

**Why:** The point of the template — push to a `site-<slug>` repo's main and get an image on `ghcr.io/optidigi/site-<slug>:latest` (and `:sha-<short>`).

- [ ] **Step 1: Create the workflow**

Run: `mkdir -p /home/shimmy/Desktop/env/sandbox/sitegen-template/.github/workflows`

Create `/home/shimmy/Desktop/env/sandbox/sitegen-template/.github/workflows/publish.yml`:

```yaml
name: publish

on:
  push:
    branches: [main]
  workflow_dispatch: {}

permissions:
  contents: read
  packages: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha,prefix=sha-,format=short

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            SITE_URL=https://${{ github.event.repository.name }}
```

- [ ] **Step 2: Lint the YAML**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/publish.yml')); print('ok')"
```
Expected: `ok`. (If python3 yaml isn't available, eyeball the file — indentation must be 2 spaces, no tabs.)

- [ ] **Step 3: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git add -A
git commit -m "feat: add GHA workflow to publish image to ghcr.io on push"
```

---

## Task 12: Template README

**Files:**
- Create/replace: `/home/shimmy/Desktop/env/sandbox/sitegen-template/README.md`

**Why:** Future you (and the agent) need a one-shot reference for what this template ships with.

- [ ] **Step 1: Replace `README.md`**

Replace `/home/shimmy/Desktop/env/sandbox/sitegen-template/README.md`:

```markdown
# sitegen-template

Astro 5 + Tailwind 4 boilerplate consumed by `optidigi/sitegen-orchestrator`.
Per-engagement, the orchestrator copies this template into `sandbox/site-<slug>/`,
integrates a theme from `sitegen-themes/`, fills content via subagents, and pushes.

## What's in the box

- **Astro 5.x**, `output: static`, with `@astrojs/sitemap` + `astro-seo`
- **Tailwind 4** via `@tailwindcss/vite` + `@tailwindcss/typography`
- **SEO baseline**: per-page `<title>`/meta/OG/Twitter, sitemap, dynamic `robots.txt`,
  `llms.txt`, `humans.txt`, `/.well-known/security.txt`, JSON-LD `Organization`
  (always) + `LocalBusiness` (if NAP supplied), favicon set, manifest
- **ContactForm** component: mailto fallback by default; renders Web3Forms
  POST form when `PUBLIC_WEB3FORMS_KEY` is set
- **Dockerfile**: multi-stage `node:lts-alpine` → `nginx:alpine`, ~30MB final image
- **nginx.conf**: gzip, asset/HTML cache strategy, security headers (CSP, X-Frame-Options, etc.)
- **GHA `publish.yml`**: push to `main` → `ghcr.io/<owner>/<repo>:latest` + `:sha-<short>`

## Local development

```bash
pnpm install
pnpm dev          # http://localhost:4321
pnpm build        # static output to dist/
pnpm astro check  # type / schema check
```

## Environment variables

See `.env.example`. `SITE_URL` is the only one the build needs;
`PUBLIC_WEB3FORMS_KEY` and `PUBLIC_CONTACT_EMAIL` are optional.

## Don't edit this template directly during a site engagement

The orchestrator copies the template into `sandbox/site-<slug>/` and works there.
Edits to this template apply to *all future* sites. Land them in a PR and pull
into `sitegen-template/` on disk before starting the next engagement.
```

- [ ] **Step 2: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
git add README.md
git commit -m "docs: add template README"
```

---

## Task 13: Orchestrator content — CLAUDE.md

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/CLAUDE.md`

**Why:** Auto-loaded every session in this dir. Keeps the high-level conventions in front of the agent without it having to read `preflight.md` first.

- [ ] **Step 1: Create `CLAUDE.md`**

Create `/home/shimmy/Desktop/env/sandbox/CLAUDE.md`:

```markdown
# sitegen-orchestrator — Claude Code conventions

You are operating the **sitegen** workflow: producing one-pager / multi-page Astro landing
pages for clients. Org is `optidigi`. New site repos: `site-<slug>`, public, images on `ghcr.io`.

## Repos in this workspace

- `sandbox/` — this orchestrator. Holds CLAUDE.md, preflight.md, prompt.md, .claude/.
- `sandbox/sitegen-template/` — Astro boilerplate (cloned, gitignored from orchestrator). Source for new sites.
- `sandbox/sitegen-themes/` — theme building blocks under `astro/<name>/` and `plain/<name>/` (cloned, gitignored).
- `sandbox/site-<slug>/` — ephemeral working copy per engagement. Created from template, deleted after deploy.

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

| Lighthouse Performance (mobile) | ≥75 |
| Lighthouse Accessibility | ≥85 |
| Lighthouse Best Practices | ≥85 |
| Lighthouse SEO | ≥95 |
| axe-core | no critical / serious violations |
| Security headers | CSP, X-Frame-Options, X-Content-Type-Options present |

See the auditor's report and address must-fix items before requesting user sign-off.

## Re-engagements

If client wants changes weeks later, `git clone optidigi/site-<slug>` back into
`sandbox/site-<slug>/`, edit, push. Skip Phases 1–3, jump to whichever phase applies.
```

- [ ] **Step 2: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
git add CLAUDE.md
git commit -m "feat: add CLAUDE.md with sitegen conventions"
```

---

## Task 14: Orchestrator content — preflight.md

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/preflight.md`

**Why:** Background context the agent reads on demand at the start of each new-site run. Drives the "summarize back what you understood" gate before `prompt.md` is opened.

- [ ] **Step 1: Create `preflight.md`**

Create `/home/shimmy/Desktop/env/sandbox/preflight.md`:

```markdown
# Preflight — read before starting a site engagement

This document gives you the context you need to run the sitegen workflow safely
and well. After reading, summarize back to the user what you understood.
**Do not open `prompt.md` until the user confirms.**

## Purpose & business model

We sell cheap, quick, quality landing pages to small businesses. Target: sites
that meet UX + SEO standards, mobile-first, responsive. Either one-pagers or up
to ~5-page sites. Speed matters — quote → live should be hours not weeks.

Each engagement produces one site with its own GitHub repo and Docker image.
The user runs the operational side (VPS, DNS, TLS); your scope ends at "image
published to ghcr.io and user confirmed deploy".

## The deploy chain end-to-end

```
sitegen-template (boilerplate) ─┐
                                ├─→ site-<slug>/ (your working copy) ─→ optidigi/site-<slug> repo
sitegen-themes (theme used)   ─┘                                           │
                                                                            │  push to main
                                                                            ↓
                                                  GitHub Actions (publish.yml)
                                                                            │
                                                                            ↓
                                                  ghcr.io/optidigi/site-<slug>:latest
                                                                            │
                                                                            ↓  (out of your scope)
                                                  VPS docker compose pull & up → live site
```

## Standards (always on)

- **SEO baseline:** per-page title/description/OG/Twitter, `sitemap.xml`,
  `robots.txt` (points to sitemap), `llms.txt`, `humans.txt`,
  `/.well-known/security.txt`, JSON-LD `Organization` (always) and
  `LocalBusiness` (if NAP supplied), correct `lang` on `<html>`, favicon set,
  `apple-touch-icon`, `manifest.json`.
- **Accessibility floor:** WCAG 2.2 AA. No critical/serious axe violations.
- **Performance budget:** Lighthouse mobile ≥75 perf, ≥85 a11y, ≥85 BP, ≥95 SEO.
- **Security headers** (set in nginx.conf): CSP, X-Content-Type-Options,
  X-Frame-Options, Referrer-Policy, Permissions-Policy.
- **Mobile-first**, fully responsive across viewports.

## Tool inventory

- `gh` — authenticated on this device. Use for repo create + GHA watch.
- `pnpm` — package manager. `pnpm install`, `pnpm build`, `pnpm dev`.
- `docker` — local container build / smoke test before push.
- `node` ≥ 20.
- `sharp` — image optimization (Astro built-in pipeline + standalone in Phase 4).
- `lhci` (Lighthouse CLI) and `@axe-core/cli` — used by `auditor` subagent.
- MCPs: `context7` (auto-loaded by plugin) for library docs lookups.

## Subagents (dispatch via the Agent tool)

- **`copywriter`** — Phase 4. Input: raw client text + page slug + role +
  keywords + tone + language. Output: writes
  `src/content/pages/<slug>.md` with SEO frontmatter and structured body.
  Hard rule: respects "do not rewrite" flag; never invents business facts.

- **`auditor`** — Phase 6, after `pnpm dev` is running. Input: preview URL +
  page paths. Runs Lighthouse mobile + axe + curl header check + curl baseline
  files check. Output: prioritized markdown report (must-fix / should-fix /
  nice-to-have). Doesn't modify the site.

- **`reviewer`** — Phase 7, after auditor passes. Wraps `code-reviewer` agent type
  with extra context. Input: site dir + intake brief + auditor report. Reviews
  brief alignment, cleanliness (no `TODO`/`lorem ipsum`/template strings),
  links, JSON-LD validity. Doesn't modify the site.

See full contracts in `.claude/agents/*.md`.

## Repo locations & permissions

- Org: `optidigi`. All sites are public by default.
- Image registry: `ghcr.io/optidigi/site-<slug>` with tags `:latest` and
  `:sha-<short>`. Auth on push is via `GITHUB_TOKEN` from GHA — no secrets to
  manage.
- VPS-side `docker compose pull && up -d` is server-side. Don't SSH; you can
  help the user diagnose by running the published image locally.

## Anti-patterns (don't do these)

- Don't invent themes — pick from `sitegen-themes/` or ask the user for a path
  / source.
- Don't push to `site-<slug>`'s `main` until the user approves preview.
- Don't strip the SEO baseline. If a theme breaks it, fix the theme integration
  — not the baseline.
- Don't write non-trivial copy inline. Dispatch `copywriter` so the main
  context stays clean.
- Don't modify `sitegen-template/` or `sitegen-themes/` during a run. Land
  template/theme improvements in their own PRs between runs.
- Don't `rm -rf` until Phase 10 (cleanup) and only on `site-<slug>/`. Never on
  template, themes, or orchestrator.

## When you're done reading

Tell the user, in your own words: (a) what the workflow does end-to-end,
(b) what subagents are available and when each fires, (c) the quality floor
the site has to clear, (d) what's out of scope for you. Then ask permission
to read `prompt.md` and start.
```

- [ ] **Step 2: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
git add preflight.md
git commit -m "feat: add preflight.md with workflow context and standards"
```

---

## Task 15: Orchestrator content — prompt.md (the runbook)

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/prompt.md`

**Why:** The phased runbook the agent follows after preflight is confirmed. Static, never modified per-run.

- [ ] **Step 1: Create `prompt.md`**

Create `/home/shimmy/Desktop/env/sandbox/prompt.md`:

```markdown
# Sitegen runbook

You have read `preflight.md` and the user has confirmed your understanding.
Follow this runbook phase-by-phase. Each **GATE** marker is a hard stop —
do not proceed past it without the action specified.

---

## Phase 1 — Intake

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

```bash
cd ~/Desktop/env/sandbox
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

```bash
cd ~/Desktop/env/sandbox/site-<slug>
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

```bash
cd ~/Desktop/env/sandbox
rm -rf site-<slug>/
```

Confirm to the user the site is shipped, the image is at
`ghcr.io/optidigi/site-<slug>:latest`, the source repo is at
`https://github.com/optidigi/site-<slug>`, and your local working dir is
clean.

Done.
```

- [ ] **Step 2: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
git add prompt.md
git commit -m "feat: add prompt.md runbook (10-phase workflow)"
```

---

## Task 16: Subagent specs — copywriter

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/.claude/agents/copywriter.md`

**Why:** Project-scoped subagent definition. The orchestrator's `prompt.md`
references this in Phase 4.

- [ ] **Step 1: Create directory**

Run: `mkdir -p /home/shimmy/Desktop/env/sandbox/.claude/agents`

- [ ] **Step 2: Create `copywriter.md`**

Create `/home/shimmy/Desktop/env/sandbox/.claude/agents/copywriter.md`:

```markdown
---
name: copywriter
description: Use when writing or refining a single page's copy with SEO + tone awareness during Phase 4 of the sitegen runbook. Input: raw client text, page slug + role, target keywords, tone notes, language. Output: writes src/content/pages/<slug>.md.
tools: Read, Write, Edit, WebSearch, WebFetch
---

You are a focused subagent within the sitegen workflow. You write copy for
exactly one page per dispatch and then return.

## Inputs (provided in your dispatch prompt)

- **Raw client text** — verbatim. May be sparse, messy, multi-language.
- **Page slug** + **role** (`home` / `about` / `services` / `contact` / `page`).
- **Target keywords** for SEO.
- **Tone / voice notes**.
- **"Do not rewrite" flag** — if set, use the raw text verbatim and only
  restructure into headings + SEO frontmatter.
- **Site language** (`nl`, `en`, etc.).
- **Word-count guidance** per section.
- **Path to write to:** absolute path to `src/content/pages/<slug>.md`.

## Output contract

Write exactly one file: `src/content/pages/<slug>.md`. Frontmatter shape:

```yaml
---
title: <≤70 chars, search-friendly, includes brand only if natural>
description: <≤160 chars, summarizes page intent, includes a primary keyword if natural>
keywords: [<3-7 phrases>]
ogImage: <optional; omit unless page warrants its own>
role: <home | about | services | contact | page>
order: <integer for nav order; 0 for home>
---
```

Body:
- One H1 (matches page intent, NOT identical to title).
- Structured H2/H3 sections appropriate to the role:
  - **home**: hero pitch, key value props, social proof / trust, CTA.
  - **about**: brief story, team / credentials if supplied, values.
  - **services**: each service with what + who-it's-for + outcome.
  - **contact**: hours, NAP, contact form pointer, map embed pointer.
  - **page** (generic): work from the raw text's natural structure.
- Image alt-text suggestions inline as HTML comments where images go:
  `<!-- alt: short descriptive alt text in site language -->`
- No CTAs to invented endpoints — if the page needs a "Book now" link and the
  user didn't supply one, write it as `<!-- TODO: book link -->` and report it.

## Hard rules

1. **Match site language.** Do not mix Dutch and English in the same page.
2. **No invented business facts.** Don't claim certifications, prices,
   service offerings, or years-in-business that aren't in the raw text. If the
   raw text is silent and the page role demands these (e.g. services page with
   no services listed), return a `Gaps:` section in your final reply listing
   what the main agent must collect from the user.
3. **Respect "do not rewrite".** When set, preserve raw text wording —
   restructure into headings, add frontmatter, fix obvious typos only.
4. **SEO discipline:**
   - Title ≤70 chars, description ≤160 chars (hard limits — if your draft is
     over, tighten before writing).
   - Use target keywords naturally; never stuff.
   - Don't repeat the title verbatim as the H1.

## What to return to the main agent

A short message:
- "Wrote `<path>`."
- One sentence on tone/structure choices.
- A `Gaps:` bulleted list if any facts are missing.
- A `Suggested links:` list for any internal/external links the page would
  benefit from but you couldn't verify.
```

- [ ] **Step 3: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
git add .claude/agents/copywriter.md
git commit -m "feat: add copywriter subagent spec"
```

---

## Task 17: Subagent specs — auditor

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/.claude/agents/auditor.md`

**Why:** Project-scoped subagent. Runs Lighthouse + axe + curl checks in Phase 6.

- [ ] **Step 1: Create `auditor.md`**

Create `/home/shimmy/Desktop/env/sandbox/.claude/agents/auditor.md`:

```markdown
---
name: auditor
description: Use during Phase 6 of the sitegen runbook, after `pnpm dev` is running. Runs Lighthouse mobile + axe + security header check + SEO baseline file check against the preview URL. Returns a prioritized markdown report. Does not modify the site.
tools: Bash, Read
---

You are a focused subagent. You run a fixed audit pipeline against a running
preview server, classify findings, and return a markdown report. You do not
modify the site.

## Inputs (provided in your dispatch prompt)

- **Preview URL** — e.g. `http://localhost:4321`.
- **Page paths** — list of routes to test (e.g. `/`, `/about`, `/services`).

## What to run

For each page path:

1. **Lighthouse mobile audit** —
   ```bash
   npx -y lighthouse <preview><path> \
     --preset=mobile \
     --only-categories=performance,accessibility,best-practices,seo \
     --quiet \
     --chrome-flags="--headless --no-sandbox" \
     --output=json --output-path=/tmp/lh-<slugified-path>.json
   ```
   Parse the JSON, extract the four category scores (0–100 ints).

2. **axe-core** —
   ```bash
   npx -y @axe-core/cli <preview><path> --exit
   ```
   Capture the violation list (impact level + rule id + nodes count).

3. **Security headers** —
   ```bash
   curl -sI <preview><path>
   ```
   Confirm presence of: `X-Content-Type-Options`, `X-Frame-Options`,
   `Referrer-Policy`, `Content-Security-Policy`, `Permissions-Policy`.

Once across the whole site (root only):

4. **SEO baseline files** — confirm 200 status:
   - `<preview>/robots.txt`
   - `<preview>/sitemap-index.xml`
   - `<preview>/llms.txt`
   - `<preview>/humans.txt`
   - `<preview>/.well-known/security.txt`
   - `<preview>/favicon.svg` (or `favicon.ico`)
   - `<preview>/apple-touch-icon.png`
   - `<preview>/og-default.png`
   - `<preview>/manifest.json`

## Classification

| Finding                                                | Severity     |
| ------------------------------------------------------ | ------------ |
| Lighthouse Performance (mobile) <75                    | must-fix     |
| Lighthouse Performance 75–84                           | should-fix   |
| Lighthouse Performance 85–89                           | nice-to-have |
| Lighthouse Accessibility <85                           | must-fix     |
| Lighthouse Accessibility 85–94                         | should-fix   |
| Lighthouse Best Practices <85                          | must-fix     |
| Lighthouse Best Practices 85–89                        | should-fix   |
| Lighthouse SEO <95                                     | must-fix     |
| axe-core impact: critical or serious                   | must-fix     |
| axe-core impact: moderate                              | should-fix   |
| axe-core impact: minor                                 | nice-to-have |
| Missing CSP / X-Frame-Options / X-Content-Type-Options | must-fix     |
| Missing other security headers (Referrer/Permissions)  | should-fix   |
| Any baseline file returns ≠200                         | must-fix     |

## Output format

Return a single markdown report:

```markdown
# Audit report — <preview URL>

## Per-page Lighthouse (mobile)

| Page | Perf | A11y | BP | SEO |
| ---- | ---- | ---- | -- | --- |
| /    | 87   | 95   | 92 | 100 |
| /about | … | … | … | … |

## Must-fix

- [page] [category] short description (specific fix hint)
- …

## Should-fix

- …

## Nice-to-have

- …

## Baseline files

- robots.txt: 200 ✓
- sitemap-index.xml: 200 ✓
- …

## Security headers (root)

- X-Content-Type-Options: ✓
- …
```

If must-fix is empty, end with: `**Status: clean — proceed to review.**`
Otherwise: `**Status: <N> must-fix items block sign-off.**`

## Hard rules

- Never modify any file in the site.
- If Lighthouse or axe fails to run (e.g., chrome missing), report the failure
  in your output as a must-fix and stop — do not silently skip.
- Don't truncate the report — main agent needs the full data.
```

- [ ] **Step 2: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
git add .claude/agents/auditor.md
git commit -m "feat: add auditor subagent spec"
```

---

## Task 18: Subagent specs — reviewer

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/.claude/agents/reviewer.md`

**Why:** Project-scoped subagent. Final pre-push gate in Phase 7.

- [ ] **Step 1: Create `reviewer.md`**

Create `/home/shimmy/Desktop/env/sandbox/.claude/agents/reviewer.md`:

```markdown
---
name: reviewer
description: Use during Phase 7 of the sitegen runbook, after the auditor reports clean. Reviews the finished site against the orchestrator's standards and the intake brief. Does not modify the site. Wraps the code-reviewer agent type with sitegen-specific context.
tools: Read, Bash, Grep, Glob
---

You are a focused subagent. You're the final pre-push gate. Review the
finished site against the standards in `preflight.md` and against the intake
brief. You don't modify anything.

## Inputs (provided in your dispatch prompt)

- **Path to** `site-<slug>/`
- **Intake brief** (the bullet summary the user approved in Phase 1)
- **Latest auditor report**

## What to check

### Standards (from preflight.md)
- `<html lang>` matches the site's intake language.
- Every page goes through `BaseLayout` (uses `Seo.astro`).
- `JsonLdOrganization` renders on every page (check the built HTML).
- `JsonLdLocalBusiness` renders if and only if `site.ts` has NAP set.
- `robots.txt`, `sitemap-index.xml`, `llms.txt`, `humans.txt`,
  `.well-known/security.txt` all present in `dist/`.
- Security headers wired in `nginx.conf` (CSP, X-Frame-Options,
  X-Content-Type-Options at minimum).
- `astro.config.mjs` has the correct `site` URL (matches primary domain).

### Brief alignment
- Every page from the scope list exists as both a route and a content file.
- `src/content/site.ts` matches intake (brand, language, primaryDomain,
  aliases, NAP, hours, serviceArea, socials, nav).
- Brand colors from intake are wired in `tailwind.config.ts` and
  `src/styles/global.css` `@theme` block.
- Logo is referenced from intake path (placed in `public/` or `src/assets/`).
- Forms: if intake said Web3Forms, `PUBLIC_WEB3FORMS_KEY` is referenced
  somewhere appropriate (`.env.example` or a contact page); if mailto, NAP
  email is set.

### Cleanliness
- No `TODO` / `FIXME` left in the site (`grep -RIn "TODO\|FIXME" src/`).
- No `lorem ipsum` (`grep -RIn -i "lorem ipsum" src/`).
- No template placeholder strings: `Example Brand`, `example.com`,
  `Replace this`, `Welcome` (the placeholder homepage h1).
- No orphaned theme files (e.g., theme demo pages not used by the site).

### Links
- Internal links resolve (`grep -RIn 'href="/[a-z]' src/` and check each
  href maps to a page that exists).
- Obvious external links use `https://` not `http://`.

### Built-output sanity
- `dist/` exists and contains `index.html` for every page in scope.
- `dist/sitemap-0.xml` lists every page in scope.
- `cat dist/index.html | grep -c application/ld+json` ≥ 1.

## Output format

Return a markdown review:

```markdown
# Review — site-<slug>

## Blocking
- [category] short description + file path / line
- …

## Non-blocking
- [category] short description
- …

## Status

`Status: clean — ready to push.`  *(or)*
`Status: <N> blocking items must be fixed before push.`
```

## Hard rules

- Never modify the site.
- Be strict but specific — don't list "could be better" without a concrete
  fix. If you can't articulate a fix, it doesn't belong in blocking.
- A clean report ends with a single line: `Status: clean — ready to push.`
```

- [ ] **Step 2: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
git add .claude/agents/reviewer.md
git commit -m "feat: add reviewer subagent spec"
```

---

## Task 19: Permissions allowlist (.claude/settings.json)

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/.claude/settings.json`

**Why:** Pre-approve the safe, common bash calls so the workflow runs smoothly without permission-prompt friction. Destructive ops stay opt-in.

- [ ] **Step 1: Create `.claude/settings.json`**

Create `/home/shimmy/Desktop/env/sandbox/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(pnpm install)",
      "Bash(pnpm install --frozen-lockfile)",
      "Bash(pnpm add:*)",
      "Bash(pnpm build)",
      "Bash(pnpm dev)",
      "Bash(pnpm astro check)",
      "Bash(pnpm astro:*)",
      "Bash(node --version)",
      "Bash(node:*)",
      "Bash(npx -y lighthouse:*)",
      "Bash(npx -y @axe-core/cli:*)",
      "Bash(corepack:*)",
      "Bash(curl -sI:*)",
      "Bash(curl -sf:*)",
      "Bash(curl -s:*)",
      "Bash(curl -I:*)",
      "Bash(gh auth status)",
      "Bash(gh repo view:*)",
      "Bash(gh repo list:*)",
      "Bash(gh run watch:*)",
      "Bash(gh run list:*)",
      "Bash(gh run view:*)",
      "Bash(gh api:*)",
      "Bash(docker build:*)",
      "Bash(docker run --rm:*)",
      "Bash(docker stop:*)",
      "Bash(docker pull:*)",
      "Bash(docker images:*)",
      "Bash(docker ps:*)",
      "Bash(git status)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(git init:*)",
      "Bash(git branch:*)",
      "Bash(git checkout:*)",
      "Bash(git remote:*)",
      "Bash(cp -r:*)",
      "Bash(mkdir -p:*)",
      "Bash(ls:*)",
      "Bash(grep:*)",
      "Bash(rg:*)",
      "Bash(find:*)",
      "Bash(magick:*)",
      "Bash(convert:*)"
    ],
    "deny": [
      "Bash(rm -rf /:*)",
      "Bash(rm -rf ~:*)",
      "Bash(rm -rf ~/:*)",
      "Bash(rm -rf $HOME:*)",
      "Bash(git push --force:*)",
      "Bash(git reset --hard:*)"
    ],
    "ask": [
      "Bash(rm -rf:*)",
      "Bash(git push:*)",
      "Bash(gh repo create:*)",
      "Bash(gh repo delete:*)"
    ]
  }
}
```

- [ ] **Step 2: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
git add .claude/settings.json
git commit -m "feat: add permissions allowlist for sitegen workflow"
```

---

## Task 20: Slash command — /new-site

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/.claude/commands/new-site.md`

**Why:** Ergonomics. `/new-site` triggers the read-preflight-then-confirm flow.

- [ ] **Step 1: Create directory and command**

Run: `mkdir -p /home/shimmy/Desktop/env/sandbox/.claude/commands`

Create `/home/shimmy/Desktop/env/sandbox/.claude/commands/new-site.md`:

```markdown
---
description: Start a new sitegen engagement (reads preflight.md, awaits confirm, then prompt.md)
---

Start a new sitegen engagement.

1. Read `preflight.md` in the current working directory.
2. Summarize back to me, in your own words: what the workflow does end-to-end,
   what subagents are available and when each fires, the quality floor a
   site has to clear, and what's out of scope for you.
3. Wait for me to explicitly confirm your understanding.
4. Once I confirm, read `prompt.md` and begin Phase 1 (Intake).

Do not skip the confirmation gate.
```

- [ ] **Step 2: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
git add .claude/commands/new-site.md
git commit -m "feat: add /new-site slash command"
```

---

## Task 21: .mcp.json (project-scoped MCP doc)

**Files:**
- Create: `/home/shimmy/Desktop/env/sandbox/.mcp.json`

**Why:** Documents which MCPs the workflow assumes are available. Currently minimal — `context7` is plugin-loaded so no config is required.

- [ ] **Step 1: Create `.mcp.json`**

Create `/home/shimmy/Desktop/env/sandbox/.mcp.json`:

```json
{
  "mcpServers": {}
}
```

- [ ] **Step 2: Commit**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
git add .mcp.json
git commit -m "chore: add empty .mcp.json (context7 is plugin-loaded)"
```

---

## Task 22: End-to-end smoke test (template only — no real client)

**Files:** none created; pure verification.

**Why:** Before pushing the three repos to `optidigi`, prove the template builds + the Docker image runs + the audit pipeline runs against the placeholder homepage. If any of these fail, fix before pushing.

- [ ] **Step 1: Clean build**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
rm -rf dist node_modules
pnpm install --frozen-lockfile
SITE_URL=https://example.com pnpm build
```
Expected: lockfile install succeeds, build succeeds.

- [ ] **Step 2: Verify SEO baseline files in dist**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
for f in robots.txt sitemap-index.xml llms.txt humans.txt .well-known/security.txt manifest.json favicon.svg apple-touch-icon.png og-default.png 404.html index.html; do
  test -f dist/$f && echo "✓ $f" || echo "✗ MISSING: $f"
done
```
Expected: all 11 files present.

- [ ] **Step 3: Verify HTML has lang, title, JSON-LD, OG**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
grep -E 'lang="nl"' dist/index.html >/dev/null && echo "✓ lang=nl"
grep -E '<title>.*\|.*</title>' dist/index.html >/dev/null && echo "✓ title brand suffix"
grep -E 'application/ld\+json' dist/index.html >/dev/null && echo "✓ JSON-LD"
grep -E 'property="og:' dist/index.html >/dev/null && echo "✓ OG tags"
grep -E 'name="twitter:' dist/index.html >/dev/null && echo "✓ Twitter tags"
```
Expected: all 5 ✓.

- [ ] **Step 4: Docker build + run smoke**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
docker build --build-arg SITE_URL=https://example.com -t sitegen-template:smoke .
docker run --rm -d -p 18080:80 --name sitegen-smoke sitegen-template:smoke
sleep 1
curl -sf -o /dev/null -w 'root: HTTP %{http_code}\n' http://127.0.0.1:18080/
curl -sf -o /dev/null -w 'robots: HTTP %{http_code}\n' http://127.0.0.1:18080/robots.txt
curl -sf -o /dev/null -w 'sitemap: HTTP %{http_code}\n' http://127.0.0.1:18080/sitemap-index.xml
curl -sf -o /dev/null -w 'llms: HTTP %{http_code}\n' http://127.0.0.1:18080/llms.txt
curl -sf -o /dev/null -w '404 page: HTTP %{http_code}\n' http://127.0.0.1:18080/does-not-exist
curl -sI http://127.0.0.1:18080/ | grep -iE '^(content-security-policy|x-frame-options|x-content-type-options|referrer-policy|permissions-policy):' | wc -l
docker stop sitegen-smoke
docker image rm sitegen-template:smoke
```
Expected:
- root, robots, sitemap, llms all return 200.
- missing path returns 404.
- 5 security headers present (the `wc -l` line should print `5`).

- [ ] **Step 5: Record image size**

Run: `docker images | grep sitegen-template || echo "(image already removed — re-run step 4 first if you want size)"`
Expected: image size <50MB.

- [ ] **Step 6: Lighthouse against the dev server (optional sanity check)**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
pnpm dev &
DEV_PID=$!
sleep 4
npx -y lighthouse http://127.0.0.1:4321/ \
  --preset=mobile \
  --only-categories=performance,accessibility,best-practices,seo \
  --quiet \
  --chrome-flags="--headless --no-sandbox" \
  --output=json --output-path=/tmp/lh-template-smoke.json || true
kill $DEV_PID 2>/dev/null || true
node -e "
  const r = require('/tmp/lh-template-smoke.json');
  const cats = r.categories;
  console.log('Performance:', Math.round(cats.performance.score*100));
  console.log('Accessibility:', Math.round(cats.accessibility.score*100));
  console.log('Best Practices:', Math.round(cats['best-practices'].score*100));
  console.log('SEO:', Math.round(cats.seo.score*100));
"
```
Expected: all four scores ≥85 on a placeholder homepage. If anything is below the floor on this trivial page, fix before pushing — the floors are even harder to hit on real sites.

(Skip this step if Chrome / Chromium isn't available locally; the GHA + per-site auditor will catch issues either way.)

---

## Task 23: Create GitHub repos and push (USER GATE)

**Files:** none created; pure remote actions.

**Why:** Publish the three repos to `optidigi`. This is the only step that has remote consequences; all preceding work has been local commits.

**This task is destructive in the sense that public repos are visible immediately. Do NOT run without explicit user approval.**

- [ ] **Step 1: Show user the plan**

Tell the user, verbatim:

> "Ready to publish three new public repos under `optidigi`:
>  - `optidigi/sitegen-orchestrator` (this dir)
>  - `optidigi/sitegen-template` (Astro boilerplate)
>  - `optidigi/sitegen-themes` (theme building blocks)
>
> All three will be public. The template repo's first push to main will
> trigger its `publish.yml` workflow, which will build and publish a Docker
> image to `ghcr.io/optidigi/sitegen-template:latest`. (Harmless — it's just
> a reusable base, the per-site repos are what matter.)
>
> Confirm to proceed with all three, or tell me which to skip."

**GATE:** Wait for explicit user approval before any of the next steps.

- [ ] **Step 2: Create + push `sitegen-orchestrator`**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox
gh repo create optidigi/sitegen-orchestrator --public --source=. --remote=origin --push
gh repo view optidigi/sitegen-orchestrator --web 2>/dev/null || true
```
Expected: repo created, all commits pushed, URL printed.

- [ ] **Step 3: Create + push `sitegen-template`**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-template
gh repo create optidigi/sitegen-template --public --source=. --remote=origin --push
gh run watch --exit-status || echo "(workflow run failed or didn't start; check gh run list)"
```
Expected: repo created, push triggers `publish.yml`, workflow finishes green, image lands at `ghcr.io/optidigi/sitegen-template:latest`.

- [ ] **Step 4: Create + push `sitegen-themes`**

Run:
```bash
cd /home/shimmy/Desktop/env/sandbox/sitegen-themes
gh repo create optidigi/sitegen-themes --public --source=. --remote=origin --push
```
Expected: repo created, push succeeds. (No workflow on this repo.)

- [ ] **Step 5: Verify all three are live**

Run:
```bash
gh repo list optidigi --limit 20 | grep -E 'sitegen-(orchestrator|template|themes)'
```
Expected: all three appear.

- [ ] **Step 6: Final report to user**

Tell the user:

- Three repos live: links to each.
- `sitegen-template`'s first GHA run: status + image path.
- The orchestrator is now ready. Next time they want a site, run from this
  dir, type `/new-site`, and walk the workflow.

---

## Self-Review

**Spec coverage:** Each section of the spec maps to tasks:

| Spec section | Implemented in |
|---|---|
| Architecture (4 repos) | Tasks 1, 2, 3 (init each repo); 23 (publish) |
| Sandbox layout | Tasks 1, 13–21 (orchestrator files), 23 (push) |
| Repo nesting strategy | Task 1 step 2 (`.gitignore`) |
| sitegen-template stack | Tasks 3, 4 |
| sitegen-template directory shape | Tasks 5–11 |
| Template conventions (SITE_URL env, site.ts data, BaseLayout wiring, ContactForm fallback, nginx.conf hardening) | Tasks 4, 5, 7, 9, 10 |
| Dockerfile (multi-stage → nginx:alpine) | Task 10 |
| GHA publish.yml (push to main, ghcr.io, latest + sha-short) | Task 11 |
| SEO baseline (always on) | Tasks 6, 7, 8 |
| Optional SEO (LocalBusiness only when NAP) | Task 6 step 3 (renders conditionally) |
| Plausible / analytics (off by default) | Out of scope of template; documented in `prompt.md` Phase 1 (Task 15) |
| CLAUDE.md | Task 13 |
| preflight.md | Task 14 |
| prompt.md (10 phases) | Task 15 |
| Intake checklist (Phase 1) | Task 15 (embedded in `prompt.md`) |
| .mcp.json | Task 21 |
| .claude/settings.json | Task 19 |
| .claude/commands/new-site.md | Task 20 |
| README.md (orchestrator) | Task 1 step 3 (basic setup info) |
| Subagent: copywriter | Task 16 |
| Subagent: auditor (incl. thresholds) | Task 17 |
| Subagent: reviewer | Task 18 |
| Error handling table | Embedded in `prompt.md` phase descriptions (Task 15) |
| State across runs | `prompt.md` (Task 15); `site-<slug>/` is gitignored persistently (Task 1) |
| Re-engagements | `CLAUDE.md` (Task 13) and `prompt.md` notes |
| Out of scope | `preflight.md` (Task 14) and `CLAUDE.md` (Task 13) |
| End-to-end smoke test | Task 22 |

No spec items uncovered.

**Placeholder scan:** No `TBD` / `TODO` / "fill in details" / "add appropriate error handling" / "similar to Task N" patterns. Every code step has the actual content.

**Type consistency:**
- `SiteConfig` shape (Task 5) is consumed by `JsonLdOrganization`/`JsonLdLocalBusiness` (Task 6), `BaseLayout` (Task 7), `ContactForm` (Task 9). All references match: `site.brand`, `site.language`, `site.primaryDomain`, `site.description`, `site.socials`, `site.nap.{displayName,legalName,street,postalCode,city,country,phone,email}`, `site.hours[].dayOfWeek/opens/closes`, `site.serviceArea`, `site.nav[]`. Consistent.
- `pages` content collection schema (Task 5) — `title`, `description`, `keywords`, `ogImage`, `role`, `order` — referenced consistently in `index.astro` (Task 7) and `copywriter` (Task 16). The copywriter spec also lists `keywords`, `role`, `order` in its frontmatter shape, matching the schema.
- Lighthouse thresholds: spec table, `CLAUDE.md` quality floors (Task 13), `auditor.md` classification (Task 17), `preflight.md` standards (Task 14) — all show `Perf ≥75, A11y ≥85, BP ≥85, SEO ≥95`. Consistent.
- Image registry path: `ghcr.io/optidigi/site-<slug>` referenced consistently across spec, plan tasks 11/15/19/23.

No inconsistencies.
