# Solomedia / Horizon — agency base theme (fork of Shopify/horizon)

## What this repo is

This is **Solomedia's fork of Shopify's Horizon theme** (`origin` = `Solomedia/horizon`, `upstream` =
`Shopify/horizon`). It is **not a client store**: it is the base theme the agency starts new client
projects from. Its purpose is to ship Horizon plus a small set of agency-wide improvements — mostly
**additional theme-editor settings** — so that future builds configure behaviour instead of writing
custom code. Every change here must therefore be:

- **Additive and neutral by default** — every new setting defaults to the current Horizon behaviour, so a
  store built from this fork renders exactly like stock Horizon until someone configures it.
- **Small and localized** — this fork is kept in sync with upstream Horizon; every customized file is a
  future merge conflict. Keep changes adjacent to the setting/logic they extend.
- **Merchant-facing, not project-specific** — nothing from one client's Figma belongs here.

Current version: **Horizon 4.1.5** (see `config/settings_schema.json → theme_info`). `main` tracks upstream
releases (merged via Shopify's `horizon-update-*` PRs); agency work lands through feature branches + PR.

### Agency customizations (keep this list current)

| Change | Files | Since |
|---|---|---|
| Mobile font sizes: `font_size_mobile` on every block that exposes `font_size` (≤749px override, default = same as desktop); `type_font_primary_size_mobile` on the header menu block (drawer top-level items); `localization_font_size_mobile` on the header section (drawer selector); `heading_size`/`heading_size_mobile` and `link_size`/`link_size_mobile` on the footer `menu` block — see [docs/mobile-font-size.md](docs/mobile-font-size.md) | 16 block schemas, `sections/header.liquid`, `snippets/typography-style.liquid`, `snippets/text.liquid`, `snippets/menu-font-styles.liquid`, `snippets/header-drawer.liquid`, `assets/base.css` (`.custom-font-size` media query) | 2026-09 |

Feature docs live in `docs/` (template: `docs/_TEMPLATE.md`). Every new setting or block added to this fork
gets one.

### Syncing with upstream Horizon

```bash
git fetch upstream
git checkout -b chore/no-ticket-horizon-update-vX.Y.Z main
git merge upstream/main        # resolve conflicts in the customized files listed above
shopify theme check
# open a PR to main — never merge upstream directly into main
```

Expect conflicts in the files from the customizations table. Re-apply the agency change on top of the
upstream version; never drop an upstream change to keep ours.

## Stack & architecture

- **Base:** Shopify Horizon (first-party theme). Theme-blocks architecture: 95 `blocks/` (private blocks
  are prefixed `_`), 42 `sections/`, 145 `snippets/`, JSON templates in `templates/`. Sections are mostly
  thin hosts that accept nested theme blocks (`@theme`, `@app`).
- **Rendering:** server-rendered Liquid; Section Rendering API for partial updates (cart, filters, variant
  picker). `layout/theme.liquid` renders `snippets/stylesheets`, `fonts`, `scripts`,
  `theme-styles-variables`, `color-palette`.
- **CSS:** global stylesheet `assets/base.css` + scoped `{% stylesheet %}` tags inside ~154 blocks/sections/
  snippets. Layout/spacing utilities come from `snippets/theme-styles-variables.liquid` (CSS custom
  properties generated from `settings`). Mobile breakpoint used across the theme: `@media screen and
  (max-width: 749px)`; secondary: `1200px`.
- **JS:** ES modules via an `importmap` in `snippets/scripts.liquid`; 76 Light-DOM custom elements in
  `assets/*.js` (`customElements.define`). Shared base in `assets/component.js`, `utilities.js`, `events.js`.
  No bundler; files are shipped as-is.
- **Colours:** Horizon 4.x uses a **global `color_palette`** (`settings_schema.json`) and **no
  `color_scheme_group`**. Do not add `color_scheme` settings — the theme has nothing that consumes them.
  Sections expose their own colour settings (background/foreground) where needed.

## Typography system (read before touching text blocks)

- Global sizes: `settings.type_size_h1…h6` and `type_size_paragraph` → `snippets/theme-styles-variables.liquid`
  emits `--font-h*--size` / paragraph vars. Sizes **≥48px scale fluidly** with `clamp()`; smaller sizes are
  fixed `rem`. There is no global mobile scale.
- Per-block typography: blocks expose `type_preset` (`rte`, `paragraph`, `h1…h6`, `custom`) and, under
  `custom`, `font`, `font_size`, `weight`, `line_height`, `letter_spacing`, `case`, `wrap`.
- **Single conversion point:** `snippets/typography-style.liquid` turns those settings into inline CSS
  variables (`--font-size`, `--font-weight`, `--font-family`, `--line-height`, `--letter-spacing`).
  `--font-size` is consumed by `.custom-font-size, .custom-font-size > *` in `assets/base.css` for every
  block that routes through that class, plus direct reads in `blocks/footer-policy-list.liquid`'s own
  stylesheet (`.policy-list-trigger`, `.policy_list li a`). Blocks add the `custom-font-size` class themselves
  when `font_size` (or `font_size_mobile`) is set.
- Text-type blocks render through `snippets/text.liquid` (text, _heading, product-title, product-description,
  collection-title, _product-list-text, _blog-post-title, _featured-blog-posts-title); price, sku,
  _announcement, footer-copyright, footer-policy-list, _blog-post-description, _inline-text,
  _inline-collection-title call `typography-style` directly.
- **Mobile override (agency):** `font_size_mobile` (select, same options as `font_size`, `""` = inherit
  desktop). `typography-style` emits `--font-size-mobile`; `base.css` applies
  `font-size: var(--font-size-mobile, var(--font-size))` at ≤749px. The label reuses `t:settings.size_mobile`,
  a stock Horizon locale key that had no consumer in the theme before this change. `blocks/footer-policy-list.liquid`
  additionally overrides its own `.policy-list-trigger` rule at ≤749px because that block sizes its trigger in
  its own stylesheet instead of through `.custom-font-size`. Full doc: `docs/mobile-font-size.md`.
- **Header typography is a separate system.** `blocks/_header-menu.liquid` sizes menu items through
  `snippets/menu-font-styles.liquid` / `submenu-font-styles.liquid` (`--menu-top-level-*`, `--menu-parent-*`,
  `--menu-child-*` variables and the `--menu-font-{sm,md,xl,2xl}--size` tokens); the desktop setting
  `type_font_primary_size` only applies at ≥750px, the drawer uses tokens. The agency `type_font_primary_size_mobile`
  overrides the drawer's top-level size at ≤749px. `sections/header.liquid` has its own `localization_font_size`
  (agency `localization_font_size_mobile` drives the drawer selector at ≤749px) and `actions_font_size` (text
  actions are hidden on mobile, so it has no mobile variant).

## Conventions

- Schema labels/options/content always through locale keys: `t:settings.*`, `t:options.*`, `t:content.*`,
  `t:names.*` in `locales/*.schema.json` (57 locales — a new key must be added to `en.default.schema.json`
  at minimum; prefer reusing an existing key).
- Conditional settings use `"visible_if": "{{ block.settings.x == 'y' }}"`.
- Existing responsive-setting pattern to copy: `blocks/spacer.liquid`, `blocks/logo.liquid` — a `*_mobile`
  setting consumed through a `--*-mobile` variable inside `@media screen and (max-width: 749px)`. (Their labels
  are `t:settings.custom_mobile_size` / `t:settings.unit`; only the pattern is shared, not the keys.)
- Block files starting with `_` are private (only usable via `content_for 'block'` or a parent block);
  public blocks are exposed in the editor.
- Commit scope for typography work: `feat(typography): …`; theme-wide chores: `chore: …`.

## Gotchas

- `main` may include upstream code for features not yet released on Shopify (see README). Test on a
  dev store on the current Shopify version before assuming a Liquid property is available.
- `snippets/text.liquid` derives the text `type` (display/heading/body → line-height & letter-spacing
  tokens) from the **desktop** `font_size`; a mobile size does not change that.
- `custom-font-size` is only added when `font_size != ""`. Any new size-related setting must extend that
  condition or it will silently do nothing.
- There is no `.theme-check.yml` in the repo: Theme Check runs with the CLI defaults. Keep it green
  (`shopify theme check`).
- **New locale keys cost 21+ files.** Theme Check's `MatchingTranslations` flags any key added to
  `en.default.schema.json` that is missing from the other `*.schema.json` files. Prefer reusing an existing key
  (`t:settings.size_mobile`, `t:options.default`, …); if a new key is unavoidable, add it to every schema locale.
- **Don't run `sed -i` inside theme folders while `shopify theme dev` is running (Windows).** sed writes a
  temporary file (`blocks/sedXXXXXX`) next to the target; the watcher uploads it, Shopify rejects it ("Must have
  a .liquid file extension") and the dev server keeps serving a 500 "Failed to Upload Theme Files" page until
  you restart `shopify theme dev`. Edit with the editor tools or write to a temp dir and move the file.

## Team workflow (Solomedia — Shopify)

This project follows the team kit. These rules are non-negotiable and must be respected every session.

**Idioma — háblale al dev SIEMPRE en español.** Toda la comunicación con el desarrollador va en
español: narración de lo que estás haciendo, resúmenes, planes presentados, preguntas, reportes de QA/
auditoría, hallazgos, y explicaciones de errores. El equipo es hispanohablante y necesita entender qué
está pasando en la consola. Solo cambia a otro idioma si el dev lo pide explícitamente. **Excepción
(convención del equipo): los artefactos técnicos siguen en inglés** — código, variables, comentarios de
código, commits, ramas, PRs (título y cuerpo) y docs en `docs/`. **Lo que va al ticket es en español**:
el comentario de cierre, los comentarios de QA y la narración del Loom (los lee el PM/equipo). Las
plantillas del kit (📋 FINAL REPORT, 🗺️ DEV PLAN, ✅ DONE, etc.) definen **estructura, no idioma** —
renderízalas al dev en español (traduce etiquetas y prosa; conserva emojis, campos, rutas y valores
técnicos). Si una herramienta imprime salida en inglés (Theme Check, git, npm…), explícale al dev en
español qué significa — no la dejes sin traducir cuando sea relevante.

**Skills — use them proactively; do NOT wait for the dev to name them.** You (Claude) are responsible
for reaching for the right skill the moment the situation calls for it. If a dev forgets to invoke one,
that's on you to catch — recognize the situation and use the skill anyway. Map:

| When the situation is… | Use this skill (without being asked) |
|---|---|
| Any task is starting — a ticket, a bug, a one-liner, or no ticket at all | **`ticket-start`** |
| Design work needs numbers and `.claude/project-config.json` is missing or stale | **`design-setup`** (once per project) |
| A token value is needed and none is documented, or the theme changed | **`theme-tokens`** |
| Work must not disturb the current tree, or two pieces of one ticket should progress in parallel | **`worktree`** |
| A new section/block/snippet is needed, or an existing one must be cloned/duplicated/adapted | **`new-section`** (reuse-first) |
| A section/block/snippet was just created or edited, or code needs a review/audit | **`review-section`** |
| Liquid/JSON changed — after building a section, before a commit | **`validate`** (Shopify Theme Check) |
| A feature with schema/metafields/non-obvious logic was completed | **`feature-doc`** (doc in `docs/`) |
| About to push / wrapping up | **`pre-push`** (missing-doc check) |
| A Loom/YouTube/MP4 appears in a ticket or chat | **`video-to-ticket`** |
| A task is being finished — however small | **`ticket-done`** (audit → PR → close-out) |

Recognize the intent behind the words, in English or Spanish ("clona esta sección", "revisa esto",
"ya terminé", "hay un loom en el ticket"). When in doubt whether a skill applies, prefer using it.

**Git — never break these:**
- **No direct push to a protected branch.** `main`, `master`, and `develop` are always protected; any
  others are listed in the marker below. Every change to them goes through a reviewed **PR**. Non-protected
  branches (feature/fix/etc.) can be pushed directly.

<!-- protected-branches: main -->

- No force-push to shared branches. No secrets/`.env`/generated files committed.
- Branch: `{type}/{ticket-ref}-{slug}`; commits use Conventional Commits with a `Ticket:` trailer;
  the clickable ticket URL is stored on the branch (`git config branch.<b>.ticket-url`).
- This fork has no `develop`: feature branches start from `main` and PR back to `main`.

**How we build:**
- **Reuse-first ladder:** existing sections/blocks as-is → *a named preset of an existing section* →
  add schema settings → minimal scoped code → custom (last resort). Least code possible; never break
  sections already in use. In this repo the ladder usually stops at **"add schema settings"** — that is the
  whole point of the fork.
- **Neutral defaults are mandatory here.** Every new setting's `default` must reproduce stock Horizon.
  Verify with a before/after of a stock page at mobile, tablet, and desktop.
- **Design facts are measured, not guessed.** In client projects the Figma file key, viewport widths and
  token values live in `.claude/project-config.json`. This fork has no Figma; QA uses the theme's own
  breakpoint (749px) and generic check widths.
- **If the Figma differs between breakpoints, the Figma wins.** A mobile size that differs from desktop
  is a requirement, not a design error — reach it through a mobile setting with a neutral default, never
  a CSS override on a shared text block. (This fork exists to make that setting available.)
- **Web Components: Light DOM only, no Shadow DOM.** BEM CSS scoped to the block, theme variables not
  hardcoded values. Match this theme's own patterns (`{% stylesheet %}` in the block, `base.css` for
  truly global rules).
- Verify version-sensitive Liquid/schema/APIs against current Shopify docs — don't rely on memory.

**Every ticket closes with:** manual QA on mobile/tablet/desktop, a PR, and a ticket comment carrying
the **PR link**, the **updated preview link** (pointing at the **exact page of the change** —
`/pages/about?preview_theme_id={id}`, not the home; one link per touched page), and a **Loom** (mandatory — it must show the
details of the change on the preview; short is fine for trivial fixes, but it always exists). The dev
advances the ticket status and assigns the PM.

**Global rule files (installed with the kit):** `workflow-rules.md`, `component-strategy-rules.md`,
`css-rules.md`, `liquid-rules.md`, `js-rules.md`, `performance-rules.md`, `git-rules.md`,
`design-system-rules.md` (generic).
Design system (palette/typography) is **per-project** — this fork ships Horizon defaults only.

<!-- claude-md-sync: updated=2026-09-04 commit=8b42ace -->
