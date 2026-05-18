# siab-site-orchestrator: produce CMS-ification-ready sites

**Status:** Draft for review
**Date:** 2026-05-18
**Backlog (in siab-payload):** `docs/backlog/infra/README.md` — OBS-56 (sister-repo sync), OBS-50 (auditor manifest gate — re-homed to reviewer in this spec) · `docs/backlog/features/README.md` — OBS-48 (template `siteManifest.example.json`)
**Depends on:** `siab-site-template@main` `ea3abc7` (OBS-56 rendering contract foundation shipped — template now ships `siteManifest.example.json` + `RtNodeRenderer` + role tokens)
**Blocks:** `siab-payload-orchestrator` updates (next spec) · per-tenant migrations (ami-care first, then amblast + siteinabox)

---

## 1. Context

`siab-site-orchestrator` is the workshop repo that runs the 10-phase sitegen workflow. It clones `siab-site-template/` into a new `site-<slug>/` working copy, integrates a theme from `siab-site-themes/`, runs copywriter for content, audits + reviews, publishes a static nginx image to `ghcr.io/optidigi/site-<slug>:latest`. The orchestrator's hard rule: never modify the template or themes during a run — only write into `site-<slug>/`.

`siab-payload-orchestrator` is the sister workshop that runs the `/add-cms <slug>` workflow. It clones the existing `optidigi/site-<slug>` repo, dispatches `payload-seeder` (writes pages/media/siteSettings into Payload via REST) and `site-converter` (surgically converts the static site to SSR + Payload-data-driven). For this CMS-ification to be seamless, the source site repo must satisfy payload-orchestrator's pre-flight expectations.

This spec updates `siab-site-orchestrator` so the sites it produces are CMS-ification-ready by default — no per-tenant fixup needed when payload-orchestrator runs.

### What changed upstream that motivates this work

`siab-site-template@main` (`ea3abc7`) now ships:
- `siteManifest.example.json` at repo root (per OBS-48) — minimal generic RtManifest schema example
- `src/components/cms/RtNodeRenderer.tsx` + 7 RtRoot-aware structure-only block renderers
- `src/styles/rich-text.css` + role tokens (`--font-{title,heading,text,script,serif,sans}` + `--radius-{sm,md,lg}`) declared in `global.css @theme {}` as placeholders
- README "Rich-text rendering contract" section documenting the post-CMS-ification surface

`siab-payload-orchestrator`'s `payload-seeder` (per OBS-49, which will be specced after this) is planned to read `siteManifest.json` from the cloned site repo and seed it into `Tenant.siteManifest` during Phase 4. The fallback is to read `siteManifest.example.json` from the template — but seamless CMS-ification means site-orchestrator should produce the per-tenant version.

## 2. Goals

1. **Phase 2 (scaffold)** generates a per-tenant `siteManifest.json` at the new site repo root by copying `siteManifest.example.json` (now shipped by the template). Leaves `.example.json` in place as documentation/restoration source.
2. **Phase 3 (theme integration)** explicitly maps the new role tokens — `--font-{title,heading,text,script,serif,sans}` + `--radius-{sm,md,lg}` — alongside color tokens. Without this, themes that consume the rt-* contract render with placeholder fonts (`ui-serif, Georgia, serif`).
3. **Phase 7 reviewer** gates Phase 8 sign-off on `siteManifest.json` presence at site repo root. Sites missing it can't ship to `optidigi/site-<slug>` until fixed.
4. **CLAUDE.md** documents the downstream CMS-ification contract so future operators understand what the orchestrator must produce + why.

## 3. Non-goals

- **CMS-ification logic** — that's `siab-payload-orchestrator`'s `site-converter` subagent's job. site-orchestrator stays static-mode.
- **`cms-editor.css` build outputs** — produced post-CMS-ification by site-converter. Not relevant to static-site spin-up.
- **OBS-55 deploy-time `docker cp` hook** — that's CMS-ified deployment territory (lives wherever a real deploy hook ever lands; today operator's manual step).
- **Theme-aware `blocks[]` trimming** during Phase 3 — themes don't declare expected blocks today; payload's server hook + `resolveAllowedBlocks`'s warn-and-skip cover the case.
- **Modifying `auditor.md`** — per OBS-50's intent, but re-homed to reviewer (more natural fit: auditor does runtime/preview-URL checks, reviewer does source-tree checks).
- **Modifying `copywriter.md` or `preflight.md` or `README.md`** — those aren't touched by this work.
- **Modifying `siab-site-template/` or `siab-site-themes/`** — hard rule per CLAUDE.md #3.

## 4. Architecture

### 4.1 `prompt.md` Phase 2 — add `siteManifest.json` generation

Current Phase 2 (around lines 57-77) ends with operator-facing scaffolding instructions: `cp -r siab-site-template/. site-<slug>/`, git init, edit `src/content/site.ts`, update `public/manifest.json`, update `public/.well-known/security.txt`.

**Append one new bullet** after the "Update `public/.well-known/security.txt` Canonical to the primary domain." line:

> - Generate the per-tenant siteManifest by copying the template's example:
>   ```bash
>   cp siteManifest.example.json siteManifest.json
>   ```
>   The CMS-ified version of this site (post-`siab-payload-orchestrator`) will read `siteManifest.json` to seed `Tenant.siteManifest`. The `.example.json` stays in place as documentation/restoration source. You can narrow `blocks[]` later (Phase 4) if the chosen theme/content uses fewer than all 7 page-block types.

### 4.2 `prompt.md` Phase 3 — extend theme-mapping to role tokens

Current Phase 3 (around lines 80-99) mapping line:

> For Astro themes: copy/adapt components into `src/components/<theme>/`. Map the theme's color tokens into `src/styles/global.css` `@theme` block and `tailwind.config.ts` `brand`/`accent` colors.

**Replace with**:

> For Astro themes: copy/adapt components into `src/components/<theme>/`. Map the theme's tokens into `src/styles/global.css` `@theme` block:
>
> **(a) Color tokens** (`--color-brand`, `--color-accent`, etc.) — also mirror in `tailwind.config.ts` for editor autocomplete.
>
> **(b) Font role tokens** — `--font-title`, `--font-heading`, `--font-text`, `--font-script`, `--font-serif`, `--font-sans`. The template ships placeholder values (`ui-serif, Georgia, serif` etc.) that themes override with their real font stacks.
>
> **(c) Radius role tokens** — `--radius-sm`, `--radius-md`, `--radius-lg`. Same placeholder-override pattern. The template ships `0.25rem`/`0.5rem`/`1rem` defaults.
>
> Example `@theme {}` block after theme integration:
> ```css
> @theme {
>   --color-brand: <theme primary>;
>   --color-brand-fg: <fg on brand>;
>   --color-accent: <theme accent>;
>   --color-accent-fg: <fg on accent>;
>   --font-title:   <theme display stack>;
>   --font-heading: <theme heading stack>;
>   --font-text:    <theme body stack>;
>   --font-script:  <theme script stack — optional>;
>   --radius-sm: <theme tight radius>;
>   --radius-md: <theme default radius>;
>   --radius-lg: <theme card radius>;
> }
> ```
>
> These role tokens are consumed by the `src/components/cms/*` block renderers via inline `style` props (post-CMS-ification surface). In static-mode they apply to any theme content that uses the same `var(--*)` references; themes that don't yet acknowledge the role tokens render correctly via their own `font-family`/`border-radius` declarations and the placeholders sit unused.

### 4.3 `reviewer.md` "Standards" checklist — add `siteManifest.json` gate

Current "Standards (from preflight.md)" section in `.claude/agents/reviewer.md` checks: `<html lang>`, BaseLayout usage, JsonLdOrganization rendering, JsonLdLocalBusiness conditional rendering, `robots.txt`/`sitemap-index.xml`/`llms.txt`/`humans.txt`/`.well-known/security.txt` in `dist/`, security headers in `nginx.conf`, `astro.config.mjs` `site` URL.

**Append one bullet** to that list:

> - `siteManifest.json` present at site repo root (per-tenant manifest used by `siab-payload-orchestrator` during CMS-ification). If absent, raise as blocking — Phase 2 should have generated it from `siteManifest.example.json`. Do NOT auto-fix; the operator should re-run Phase 2's cp step manually so they understand the dependency.

### 4.4 `CLAUDE.md` — add "Downstream CMS-ification contract" subsection

Append after the existing "Repos in this workspace" section:

```markdown
## Downstream CMS-ification contract

Sites produced by this orchestrator are designed to be cleanly CMS-ified by `siab-payload-orchestrator`. To stay compatible:

- **Phase 2 always generates `siteManifest.json`** at site repo root (per-tenant manifest read by payload-seeder during CMS-ification). Template ships `siteManifest.example.json` as the source.
- **Phase 3 maps theme tokens into the full role-token set** — `--font-{title,heading,text,script,serif,sans}` + `--radius-{sm,md,lg}` — not just colors. The template's `cms/*` renderers consume these via inline `style` props for the post-CMS-ification rendering surface.
- **The template ships `src/components/cms/*` renderers + `RtNodeRenderer`** for the post-CMS-ification surface. These are inert in static-mode (only invoked via the wave-3 preview-iframe) and become live when `site-converter` (in `siab-payload-orchestrator`) rewires page routes during Phase 5 of CMS-ification.

Reviewer Phase 7 gates on `siteManifest.json` presence — sites missing it can't ship.
```

### 4.5 No changes elsewhere

- `auditor.md` — unchanged (gate re-homed to reviewer)
- `copywriter.md` — unchanged (content creation doesn't touch manifest or role tokens)
- `preflight.md` — unchanged (existing deploy-chain diagram + standards already cover the orchestrator boundary)
- `README.md` — unchanged (operator setup instructions are unaffected)
- `siab-site-template/` / `siab-site-themes/` — never modified (hard rule)

## 5. Backwards compatibility

Existing static-site flow is preserved. Phase 2 gains one additional `cp` command; if it fails (e.g. `.example.json` missing from a pre-OBS-56 template clone), the script fails loudly and operator can resolve. Phase 3 gets richer guidance but doesn't change the mapping mechanism (still `@theme {}` block + `tailwind.config.ts`). Reviewer gains one new checklist item; if Phase 2 ran correctly, the gate is satisfied automatically.

Sites built from an outdated template clone (pre-`ea3abc7`) will fail Phase 2's `cp` step — operator updates their local `siab-site-template/` clone and retries. No silent failures.

## 6. Risks

- **`siteManifest.example.json` not present in template clone.** Operator's local `siab-site-template/` is stale (pre-OBS-56 merge). Mitigation: Phase 2 `cp` fails fast with a clear error; preflight could optionally check template freshness but that's out of scope here.
- **Theme integration doesn't actually override role tokens.** Sites ship with placeholder fonts (`ui-serif, Georgia, serif`). Visually wrong but not functionally broken — themes that don't acknowledge role tokens render via their own `font-family`/`border-radius` declarations, leaving the tokens unused. Reviewer doesn't currently inspect actual theme-override completeness (potential future check, not in scope here).
- **Operator manually edits `siteManifest.json` wrong** (e.g. lists a block slug that doesn't exist in `siab-payload/src/blocks/registry.ts`). Mitigation: `resolveAllowedBlocks` in siab-payload warns + skips unknown slugs at runtime per OBS-57's design. Non-fatal.
- **`feat/site-orch-cmsification-readiness` branch isn't merged before payload-orchestrator's downstream spec lands.** Mitigation: this spec explicitly lists "blocks payload-orchestrator updates" so the dependency is visible.

## 7. Acceptance criteria

- [ ] `prompt.md` Phase 2 includes the `cp siteManifest.example.json siteManifest.json` step (in scaffold)
- [ ] `prompt.md` Phase 3 mapping section explicitly mentions `--font-*` + `--radius-*` role tokens with an example `@theme {}` block
- [ ] `.claude/agents/reviewer.md` "Standards" checklist includes `siteManifest.json` presence
- [ ] `CLAUDE.md` has the "Downstream CMS-ification contract" subsection
- [ ] No changes to `auditor.md`, `copywriter.md`, `preflight.md`, `README.md`
- [ ] No accidental modification of `siab-site-template/` or `siab-site-themes/`

## 8. Sequencing — what this unblocks

Once this spec lands and the implementation merges, the next spec in the chain is `siab-payload-orchestrator` updates:

1. **`payload-seeder`** reads `siteManifest.json` from the cloned site repo (now reliably present per this spec's Phase 2 step) and writes to `Tenant.siteManifest`. Falls back to `.example.json` only if the per-tenant version is missing (resilience against operator manually deleting it).
2. **`payload-seeder`** generates `RtRoot`-shaped block content (not HTML strings) for rich-text fields per the RT v2 schema in siab-payload.
3. **`site-converter`** does the heavy SSR conversion (Dockerfile, `lib/cms.ts`, `middleware.ts`, BaseLayout tenant-theme injection, scripts/build-cms-css.mjs).

After payload-orchestrator updates land, the per-tenant migrations (ami-care, then amblast + siteinabox) become the last leg of the OBS-56 program.
