# site-orch CMS-ification readiness — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update `siab-site-orchestrator` so sites it produces satisfy `siab-payload-orchestrator`'s preflight expectations — namely, ship a per-tenant `siteManifest.json` + integrate themes that override the new role tokens.

**Architecture:** Pure doc/prompt edits. 4 files touched: `prompt.md` (Phase 2 cp step + Phase 3 role-token guidance), `.claude/agents/reviewer.md` (new presence-gate bullet), `CLAUDE.md` (new "Downstream CMS-ification contract" subsection). No code, no tests, no deps — orchestrator is a workflow-instructions repo.

**Tech Stack:** Markdown.

**Spec:** `docs/specs/2026-05-18-site-orch-cmsification-readiness-design.md`

---

## Prerequisites

```bash
cd /home/shimmy/Desktop/env/siab/siab-site-orchestrator
git status   # confirm on feat/site-orch-cmsification-readiness branch, working tree clean
```

If branch/tree state diverges, fix before proceeding.

---

## Task 1: Add `siteManifest.json` generation step to `prompt.md` Phase 2

**Files:**
- Modify: `prompt.md` (Phase 2 scaffold section, around line 75 — insert before the "Verify:" line)

- [ ] **Step 1: Read current Phase 2 to confirm context**

```bash
sed -n '57,77p' prompt.md
```

Expected: shows Phase 2 ending with:
```
- Update `public/manifest.json` (name, short_name, theme_color).
- Update `public/.well-known/security.txt` Canonical to the primary domain.

Verify: `pnpm install && pnpm astro check` succeeds.
```

- [ ] **Step 2: Apply the edit**

Insert a new bullet after the `public/.well-known/security.txt` line. Use Edit tool with this old_string / new_string:

**old_string** (exact, including the blank line after):
```
- Update `public/.well-known/security.txt` Canonical to the primary domain.

Verify: `pnpm install && pnpm astro check` succeeds.
```

**new_string**:
```
- Update `public/.well-known/security.txt` Canonical to the primary domain.
- Generate the per-tenant siteManifest by copying the template's example:
  ```bash
  cp siteManifest.example.json siteManifest.json
  ```
  The CMS-ified version of this site (post-`siab-payload-orchestrator`)
  will read `siteManifest.json` to seed `Tenant.siteManifest`. The
  `.example.json` stays in place as documentation/restoration source.
  You can narrow `blocks[]` later (Phase 4) if the chosen theme/content
  uses fewer than all 7 page-block types.

Verify: `pnpm install && pnpm astro check` succeeds.
```

- [ ] **Step 3: Verify the edit**

```bash
grep -n "siteManifest.example.json siteManifest.json" prompt.md
grep -n "narrow `blocks\[\]` later" prompt.md
```

Expected: both grep hits return a line number; the new bullet is present.

- [ ] **Step 4: Commit**

```bash
git add prompt.md
git commit -m "$(cat <<'EOF'
feat(prompt): Phase 2 generates per-tenant siteManifest.json

Adds a scaffold step that copies siteManifest.example.json from the
template into siteManifest.json in the new site repo root. Required
for seamless CMS-ification by siab-payload-orchestrator's payload-seeder
(OBS-49). The .example.json stays in place as documentation/restoration.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Extend `prompt.md` Phase 3 theme-mapping to role tokens

**Files:**
- Modify: `prompt.md` (Phase 3 theme integration, around lines 85-87)

- [ ] **Step 1: Read current Phase 3 to confirm context**

```bash
sed -n '82,98p' prompt.md
```

Expected: shows the Phase 3 mapping bullet:
```
- For Astro themes: copy/adapt components into `src/components/<theme>/`. Map
  the theme's color tokens into `src/styles/global.css` `@theme` block and
  `tailwind.config.ts` `brand`/`accent` colors.
```

- [ ] **Step 2: Apply the edit**

Use Edit tool to replace the mapping bullet with an expanded version covering all three token categories.

**old_string** (exact):
```
- For Astro themes: copy/adapt components into `src/components/<theme>/`. Map
  the theme's color tokens into `src/styles/global.css` `@theme` block and
  `tailwind.config.ts` `brand`/`accent` colors.
```

**new_string**:
```
- For Astro themes: copy/adapt components into `src/components/<theme>/`. Map
  the theme's tokens into `src/styles/global.css` `@theme` block in three
  categories:

  **(a) Color tokens** (`--color-brand`, `--color-accent`, etc.) — also
  mirror in `tailwind.config.ts` for editor autocomplete.

  **(b) Font role tokens** — `--font-title`, `--font-heading`, `--font-text`,
  `--font-script`, `--font-serif`, `--font-sans`. The template ships
  placeholder values (`ui-serif, Georgia, serif` etc.) that themes override
  with their real font stacks.

  **(c) Radius role tokens** — `--radius-sm`, `--radius-md`, `--radius-lg`.
  Same placeholder-override pattern. The template ships `0.25rem` / `0.5rem`
  / `1rem` defaults.

  Example `@theme {}` block after theme integration:
  ```css
  @theme {
    --color-brand: <theme primary>;
    --color-brand-fg: <fg on brand>;
    --color-accent: <theme accent>;
    --color-accent-fg: <fg on accent>;
    --font-title:   <theme display stack>;
    --font-heading: <theme heading stack>;
    --font-text:    <theme body stack>;
    --font-script:  <theme script stack — optional>;
    --radius-sm: <theme tight radius>;
    --radius-md: <theme default radius>;
    --radius-lg: <theme card radius>;
  }
  ```

  These role tokens are consumed by `src/components/cms/*` block renderers
  via inline `style` props (post-CMS-ification surface). In static-mode the
  tokens apply to any theme content that uses the same `var(--*)`
  references; themes that don't acknowledge the role tokens render via
  their own `font-family`/`border-radius` declarations and the placeholders
  sit unused.
```

- [ ] **Step 3: Verify the edit**

```bash
grep -n "Font role tokens" prompt.md
grep -n "Radius role tokens" prompt.md
grep -n "Example .@theme" prompt.md
```

Expected: all three grep hits return a line number — the new structured mapping is in place.

- [ ] **Step 4: Commit**

```bash
git add prompt.md
git commit -m "$(cat <<'EOF'
feat(prompt): Phase 3 maps theme role tokens (fonts + radii) alongside colors

Previously Phase 3 only documented mapping color tokens. The template
now ships placeholder values for --font-{title,heading,text,script,
serif,sans} + --radius-{sm,md,lg} as part of OBS-56's rendering
contract; themes must override these for correct visual rendering.
Adds an example @theme {} block + clarifies the role tokens are
consumed by the cms/* renderers (post-CMS-ification surface).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Add `siteManifest.json` presence gate to `.claude/agents/reviewer.md`

**Files:**
- Modify: `.claude/agents/reviewer.md` (Standards section, around line 19)

- [ ] **Step 1: Read current Standards section to confirm context**

```bash
sed -n '19,29p' .claude/agents/reviewer.md
```

Expected: shows the existing Standards checklist (BaseLayout, JsonLd, baseline files, security headers, astro.config site URL).

- [ ] **Step 2: Apply the edit**

The current Standards bullet list ends with `astro.config.mjs has the correct site URL (matches primary domain).`. Append a new bullet after it.

Find the exact ending line via:
```bash
grep -n "astro.config.mjs has the correct" .claude/agents/reviewer.md
```

Use Edit tool to append after that line.

**old_string** (exact):
```
- `astro.config.mjs` has the correct `site` URL (matches primary domain).
```

**new_string**:
```
- `astro.config.mjs` has the correct `site` URL (matches primary domain).
- `siteManifest.json` present at site repo root (per-tenant manifest used by
  `siab-payload-orchestrator` during CMS-ification). If absent, raise as
  blocking — Phase 2 should have generated it from `siteManifest.example.json`.
  Do NOT auto-fix; the operator should re-run Phase 2's `cp` step manually so
  they understand the dependency.
```

- [ ] **Step 3: Verify the edit**

```bash
grep -n "siteManifest.json" .claude/agents/reviewer.md
```

Expected: line number returned; the new bullet is present.

- [ ] **Step 4: Commit**

```bash
git add .claude/agents/reviewer.md
git commit -m "$(cat <<'EOF'
feat(reviewer): gate Phase 7 sign-off on siteManifest.json presence

Adds a Standards-checklist bullet so reviewer blocks sign-off if the
per-tenant siteManifest.json is missing from the site repo root. Per
OBS-50 re-homed to reviewer (auditor stays runtime/preview-URL focused;
reviewer already does source-tree checks).

If the gate fires, instruct operator to manually re-run Phase 2's
cp step rather than auto-fixing — keeps the dependency visible.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Add "Downstream CMS-ification contract" subsection to `CLAUDE.md`

**Files:**
- Modify: `CLAUDE.md` (after the existing "Repos in this workspace" section, around line 14)

- [ ] **Step 1: Read current CLAUDE.md structure to confirm context**

```bash
sed -n '1,16p' CLAUDE.md
```

Expected: shows the intro paragraph + "Repos in this workspace" section ending at line 13/14 with the `./site-<slug>/` bullet.

- [ ] **Step 2: Apply the edit**

Insert the new section between "Repos in this workspace" and "Subagents available".

**old_string** (exact — the boundary between sections):
```
- `./site-<slug>/` — ephemeral working copy per engagement. Created from template, deleted after deploy.

## Subagents available
```

**new_string**:
```
- `./site-<slug>/` — ephemeral working copy per engagement. Created from template, deleted after deploy.

## Downstream CMS-ification contract

Sites produced by this orchestrator are designed to be cleanly CMS-ified by `siab-payload-orchestrator`. To stay compatible:

- **Phase 2 always generates `siteManifest.json`** at site repo root (per-tenant manifest read by payload-seeder during CMS-ification). Template ships `siteManifest.example.json` as the source.
- **Phase 3 maps theme tokens into the full role-token set** — `--font-{title,heading,text,script,serif,sans}` + `--radius-{sm,md,lg}` — not just colors. The template's `src/components/cms/*` renderers consume these via inline `style` props for the post-CMS-ification rendering surface.
- **The template ships `src/components/cms/*` renderers + `RtNodeRenderer`** for the post-CMS-ification surface. These are inert in static-mode (only invoked via the wave-3 preview-iframe) and become live when `site-converter` (in `siab-payload-orchestrator`) rewires page routes during Phase 5 of CMS-ification.

Reviewer Phase 7 gates on `siteManifest.json` presence — sites missing it can't ship.

## Subagents available
```

- [ ] **Step 3: Verify the edit**

```bash
grep -n "Downstream CMS-ification contract" CLAUDE.md
grep -c "^## " CLAUDE.md
```

Expected: first grep returns one line number; second grep returns `6` (was 5 sections; now 6 with the new subsection).

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "$(cat <<'EOF'
docs(claude.md): downstream CMS-ification contract section

Documents what the orchestrator must produce so siab-payload-orchestrator
can seamlessly CMS-ify sites: per-tenant siteManifest.json (Phase 2),
full role-token mapping (Phase 3), and acknowledgment that the template
ships cms/* renderers + RtNodeRenderer for the post-CMS-ification
surface. Includes pointer to the reviewer Phase 7 gate.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Final verification + close-out

**Files:** No code changes; verification only.

- [ ] **Step 1: Verify all 4 expected changes landed**

```bash
echo "=== Phase 2 siteManifest.json step ==="
grep -n "siteManifest.example.json siteManifest.json" prompt.md
echo "=== Phase 3 role tokens ==="
grep -n "Font role tokens" prompt.md
grep -n "Radius role tokens" prompt.md
echo "=== reviewer gate ==="
grep -n "siteManifest.json" .claude/agents/reviewer.md
echo "=== CLAUDE.md subsection ==="
grep -n "Downstream CMS-ification contract" CLAUDE.md
```

Expected: all 5 greps return line numbers (none empty).

- [ ] **Step 2: Verify no unintended files changed**

```bash
git diff main --stat
```

Expected: exactly 4 files in the diff:
- `prompt.md` (~30 insertions, 3 deletions for Phase 3 replace)
- `.claude/agents/reviewer.md` (~5 insertions)
- `CLAUDE.md` (~11 insertions)
- `docs/specs/2026-05-18-site-orch-cmsification-readiness-design.md` (already committed — the spec)
- `docs/plans/2026-05-18-site-orch-cmsification-readiness-plan.md` (this file — already committed)

NO changes to `siab-site-template/` or `siab-site-themes/` (hard rule per CLAUDE.md). NO changes to `auditor.md`, `copywriter.md`, `preflight.md`, `README.md`.

- [ ] **Step 3: Verify commit count**

```bash
git log --oneline main..HEAD
```

Expected: 6 commits (spec + plan + 4 implementation commits — one per modified file).

- [ ] **Step 4: No commit needed if all verifications pass**

Task 5 is verification only. If everything green, no further commit.

If any verification fails, identify which task's edit didn't land cleanly, re-do that task's edit, commit with `fix(site-orch): <what>`, and re-run from Step 1.

---

## Done — what got built

After all 5 tasks:

1. **`prompt.md` Phase 2** — new bullet generates per-tenant `siteManifest.json` from the template's example.
2. **`prompt.md` Phase 3** — replaced color-only mapping with structured (a)/(b)/(c) mapping covering color + font role + radius role tokens, with an example `@theme {}` block.
3. **`.claude/agents/reviewer.md`** — new Standards-checklist bullet gates Phase 7 sign-off on `siteManifest.json` presence.
4. **`CLAUDE.md`** — new "Downstream CMS-ification contract" subsection documents the orchestrator's downstream obligations.

**Total estimated effort:** 5 tasks, 2-5 min each ≈ 10-25 minutes of focused work + verification.

**Downstream unlocked:**
1. `siab-payload-orchestrator` spec (next) — payload-seeder reads the now-reliably-present `siteManifest.json`, generates RtRoot seeds; site-converter does the SSR conversion.
2. Per-tenant migrations (ami-care, then amblast + siteinabox).
