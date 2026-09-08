# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Layout

```
src/            AUTHORED source — the only place to edit
  tokens/         token CSS (+ tailwind.css, generated into the source graph)
  css/            the .ns-* layer: foundation, primitives, navigation,
                  content, product, integrations
  react/          thin React wrappers — behaviour only, never styling
  icons/          icon source: <style>/<name>.svg
  patterns/       background patterns
  styles.css      entry: tokens + base + the layer
  tailwind.css    Tailwind v4 entry
dist/           GENERATED — nsds.css, nsds.min.css, the Tailwind pair,
                tokens.json / tokens.js / tokens.d.ts. Committed on purpose.
icons/          GENERATED sprite + manifest, plus the Phosphor subset
```

`src` is authored, `dist` is built. One deliberate exception:
`src/tokens/tailwind.css` is generated but lives in `src` because it is a link
in the *source* graph — `src/tailwind.css` imports it so Tailwind's `@theme`
can see the token names. Writing it to `dist` would make a source file import
a build output.

Bundles are `nsds.*` to match the package name. Consumers use the subpath
exports — `nsds-design-system/css`, `/css/min`, `/tailwind`, `/tokens.json` —
never deep `dist/` paths, so the internal layout can move again without
breaking anyone.

**Icons.** `src/icons/<style>/<name>.svg` is the source — one SVG per icon per
style (`regular`, `fill`, `duotone`), built to `icons/nsds-icons.svg` +
`icons/icons.json` by `scripts/build-icons.mjs` and gated by `--check`. Adding
an icon is dropping a file in a folder. Write the `keywords:` line in its
header comment or nobody finds it by the word they actually type. Every icon
needs a `regular`; the build fails on a variant with no default.
`docs/ICONS.md` is the guide.

**Overriding.** `docs/CUSTOMISING.md`: retheme with `--ns-*` tokens first, use
the `components` layer to beat a rule without `!important`, fork under a new
name only when the structure changes.

## Commands

```bash
npm install
npm run dev          # gulp: build → serve at :4322 → watch → SSE live reload
                     # opens the HOMEPAGE at / ; styleguide is at /preview/
npm run build        # tokens → CSS bundle → styleguide. Run after ANY token or CSS edit.
npm run check        # the ten gates CI runs (npm test is an alias)
npm run site         # build + stage _site/ + link check
npm run serve        # serve an existing build, no watching
```

The ten gates in `npm run check` are: `build-tokens --check`, `lint-principles`, `check-cascade`, `check-components`, `check-layout`, `check-icons`, `check-palette`, `build-css --check`, `build-skill --check`, `build-icons --check`. **`check-markup` and `check-links` are not among them** — run those yourself when you touch markup or add pages.

Individual gates, for a fast loop while iterating:

```bash
node scripts/lint-principles.mjs   # raw values in components/css
node scripts/check-cascade.mjs     # layer-order contract
node scripts/check-components.mjs  # .jsx parses; no inline styling
node scripts/check-layout.mjs      # no inline layout in templates/
node scripts/check-palette.mjs     # chart hues vs colourblind floors
node scripts/check-markup.mjs      # every .ns-* used is defined
node scripts/check-icons.mjs       # glyphs present in the Phosphor subset (warns, does not fail)
node scripts/check-links.mjs       # needs _site/ — run `npm run site` first
node scripts/build-tokens.mjs --check
```

Every gulp task is a thin wrapper over `scripts/*.mjs`, so `node scripts/<x>.mjs` always works with no gulp.

**Check by exit code, not by grepping output.** The linters print human-readable findings with no `FAIL`/`ERROR` prefix, so `npm run check | grep -i error` reports success on a failing build. Use `npm run check; echo $?`.

## The layout contract

Three nested levels, and one rule: **a level never does another level's job.**

```
SECTION      .ns-band          full-bleed; the ground + the space between sections
  CONTAINER  .ns-band__inner   the max width + the gutters
    STACK    .ns-stack         the rhythm BETWEEN the children
```

A band does not set a width. A container does not set a background. **An
element does not set its own top margin** — its parent's stack does, via
`> * + *`, which is also why the first child never carries a stray one.

`src/css/foundation/layout.css` holds the primitives: `.ns-stack` (four
steps), `.ns-cluster` (the horizontal counterpart, plus `__end` and `__fill`),
`.ns-center` (the container level for anything not inside a band) and
`.ns-bare` (the user-agent box removed — a reset, not a layout decision).

`check-layout.mjs` fails the build on a layout property inside a `style=`
attribute in `templates/`. Per-instance data (`--v`, `--x`, `--seg`, chart
values) stays legal, as does anything carrying `<!-- layout-ok: reason -->`.

**Before writing an inline margin, look for the class.** Half the 133
declarations this gate removed were a component that already existed and could
not be found — the blog rail hand-rolled `.ns-widget` in three places while
`blog.css` had already built it, comment and all. `docs/LAYOUT.md` is the
guide; the styleguide's **Layout & elevation** page is the live demo.

## Architecture

### One CSS layer, two products

This system feeds a **Ghost theme** (Handlebars + Tailwind v4) and a **Next.js LMS** (React). The only thing they can genuinely share is CSS, so:

- `src/css/**/*.css` — the `.ns-*` class layer. **All styling lives here**, grouped into seven folders: `foundation/` (a11y, typography, motion, icon, print — the invisible groundwork), `layout/` (**one file per layout level**: section, container, grid, stack), `components/` (**one folder per component**: button/, form/, card/, tag/, badge/, chip/, avatar/, list/, table/, code/, overlay/, feedback/, progress/, media/, strip/, marquee/, divider/, kbd/, copy/, deflist/, logo/), `navigation/`, `content/` (prose, content, sections — the marketing bands built ON the section level, blog), `product/` (lms, training, catalog, player, deck, ai, admin, helpdesk, auth, chart) and `integrations/` (ghost, ads, monetization). `index.css` is the import manifest and **its order is the cascade order** — the folder a file lives in and the position it is imported at are independent decisions, so never reorder it casually. `docs/BUILD.md` is the full pipeline walkthrough.
  - A component owns its own file; a *container* owns how a component looks inside it. `.ns-topnav__actions .ns-btn` belongs in `navigation/navbar.css`, not in `components/button/button.css` — otherwise the button has to know about the navbar, the player and the deck.
  - Scripts that read this directory must use `scripts/lib/css-files.mjs`, which walks it recursively and **throws when the walk is empty**. A flat `readdirSync` returns nothing after a move and a checker that examines zero files still passes.
- `components/<domain>/*.jsx` — thin React wrappers. Behaviour only: generated ids, ARIA wiring, roving tabindex, focus restoration. `check-components.mjs` fails the build on inline styling.
- `templates/*.html` — the framework-agnostic HTML contract for full surfaces, which each stack adapts.
- `docs/INTEGRATION.md` — the contract between the two products.

A React component that styles itself is a component the Ghost theme has to reimplement.

### Tokens are generated, and the generation is enforced

`src/tokens/*.css` is **authored**. `dist/tokens.json`, `tokens.js`, `tokens.d.ts` and `src/tokens/tailwind.css` are **generated** by `npm run build`. CI runs `git diff --exit-code` after building, so a hand-edit or a forgotten regeneration fails.

**Adding a token requires a matching Tailwind emit.** `build-tokens.mjs` has a "bridge" check that fails if a `src/tokens/*.css` variable has no corresponding `--text-*` / `--color-*` / `--space-*` utility. Add the emit line in `build-tokens.mjs` in the same commit, or the build refuses.

### The cascade contract

`styles.css` declares `@layer theme, base, ns-components, components, utilities` before any import. That single line is why a Tailwind utility always beats a `.ns-*` default with no `!important`, and why a consuming app's own `components` layer beats ours regardless of import order. Two things silently revoke it — a rule outside any layer, and a changed order — so `check-cascade.mjs` parses the `@import` graph and guards it.

Consequence: specificity inside `src/css/` only decides DS-vs-DS conflicts. A `:has()` chain still loses to `.p-4`.

### The styleguide generates itself from the real artifacts

- `scripts/component-docs.mjs` — a single `COMPONENTS` array is the **source** for every per-component page. One entry → `preview/c-<id>.html`. The `html` string in a variant both renders the demo and prints as the code sample, so they cannot disagree.
- `scripts/build-preview.mjs` — renders those into pages, plus the foundation sections, chart docs and content-creation docs, plus the full-page demos (a `DEMOS` array maps a `templates/*.html` to `preview/demo-*.html`).
- `.ns-band--dark` clips with **`overflow: clip`, not `hidden`** — and that is load-bearing. `hidden` makes the band a scroll container, which silently kills `position: sticky` (it pins to a box that never scrolls) and freezes `view()` timelines at 50%. Any new clipping container on a band should use `clip` for the same reason.
- `src/css/motion.css` — nine entrances plus `.ns-parallax`. Scroll-driven motion needs **two** classes: `.ns-parallax-frame` declares the named view timeline and must be an element whose own nearest scroll container is the page; `.ns-parallax` layers inside it consume it. A bare `view()` inside an `overflow: hidden` box resolves against a container that never scrolls and freezes at 50% — which looks exactly like no animation at all. Debugging note: a hidden document (a background tab, an automation window) deactivates every scroll timeline and stops IntersectionObserver, so "the motion is dead" is worth checking against `document.visibilityState` before it is worth debugging.
- `icons/brand-marks.svg` — placeholder logo lockups for `.ns-brandmark`. **Every company in it is fictional on purpose** (Acme, Globex, Initech, Initrode, Umbrella, Northwind, Contoso). Do not add a real company: a logo wall showing a mark its owner never granted is a fabricated endorsement. Each symbol carries its own viewBox width and the host `<svg>` must repeat it — an external `<use>` gives the host no intrinsic size.
- `**/*.card.html` — standalone specimens, discovered by walking the tree. Config lives in an HTML comment on **line 1**: `<!-- @dsCard group="…" viewport="760x1200" name="…" subtitle="…" -->`. The `viewport` height must match what the card actually renders to, or the styleguide frames it with a scrollbar or dead space — measure it in a browser rather than guessing.
- `preview/pages.json` — a manifest `build-preview.mjs` writes; `build-site.mjs` reads it to generate the homepage's link index and `sitemap.xml`. A new page appears in both automatically or in neither.

`preview/`, the root `index.html` and `_site/` are git-ignored (all are one command away). The homepage is generated by `scripts/build-home.mjs` and written twice — to the repo root for the dev server and to `_site/` for the deploy — from one `renderHome()`, so the page you develop against and the page that ships cannot drift. `dist/` **is** committed on purpose — the Ghost theme's own gulp pipeline consumes the flat bundle directly, and `build-css.mjs --check` proves it matches source.

### The type scale forks

`--size-body` (14px) is the **UI** base — rails, tables, admin, player chrome. `--size-prose` (17px) / `--size-prose-lead` (20px) / `--size-prose-small` (15px) are the **reading** scale, used by `.ns-prose` and the surfaces built on it. The axis is scanned-versus-read. Do not set an article at `--size-body` or a table row at `--size-prose`.

**Marketing bands are on the reading side of that fork.** `src/css/sections.css` takes `--size-prose-lead` for a lede, `--size-prose` for a card heading and `--size-prose-small` for body copy — it was on the UI scale once, which put homepage paragraphs at 13px, one step *below* the base and level with a table caption. Nobody scans a homepage; they read it once, deciding. The mono label voice (`--size-label`) does not move on either side of the fork: a kicker, an index, a duration and a stat label are data, not prose.

A card composed **into** a band follows the band (rule at the foot of `sections.css`); the same card in a catalogue grid or a rail keeps the UI scale. That is the fork applied, not an exception to it — three courses on a homepage are read, forty in a catalogue are scanned. `.ns-lesson` is deliberately excluded, because "a lesson row looks the same everywhere" is a promise `lms.css` makes.

**Mono is for values; sans is for words.** The label voice (`--font-mono`, uppercase, tracked, `--size-label`) belongs on things read as data — an index, a duration, a count, a price, a timestamp, a status. It does not belong on a person's name, a stat's label, a call to action or a sentence of fine print. The test: would you say it aloud as a word, or read it off as a figure? A name in the label voice reads as a serial number. The band kicker is the one phrase that keeps the mono voice, at `--size-mono` (13) rather than the 11px label floor.

Leading forks with it: `--leading-prose` (1.7) is for prose paragraphs only, `--leading-body` (1.6) is the UI standard, `--leading-snug` (1.45) is for multi-line scanned text (card excerpts, TOC entries, 20px ledes/blockquotes). Leading grows with the measure and shrinks with the size — never put 1.7 on a table row or 1.3 on a wrapping sentence.

## Rules the build enforces

`lint-principles.mjs` fails on any raw color, radius, `z-index`, transition duration, `font-family` or padding/margin/gap literal inside `src/css/`, plus any border wider than 3px.

**It also catches raw values inside `var()` fallbacks** — `var(--duration-slow, 240ms)` fails. Use the token with no fallback.

If a value genuinely cannot be a token, append `/* lint-ok: <reason> */` on that line, so the exception is argued for in the diff rather than buried in a config.

`check-markup.mjs` requires every `.ns-*` class appearing in any `.html`, `.jsx` or `.mjs` to have a rule in the component layer. Inventing a class name in a doc entry or a template without adding the CSS fails the build.

Before adding a component, `docs/CONTRIBUTING.md` asks three questions in order: does the platform already do this (restyle `<dialog>`/`<details>`/`<progress>`, do not reimplement); do **both** products need it; does it pass the five Design Principles in `readme.md`. The accessibility floor in that file is a requirement list, not aspirations.

Do not hand-tune chart hues. The palette was solved against six computable checks and a "slightly warmer" nudge can drop deuteranopia separation below the floor with no visible change — re-step the hue until `check-palette.mjs` passes.

## Deploy

`main` → GitHub Actions → GitHub Pages at <https://sfdc.imswarnil.com/>. `gulp site` stages `_site/`, and `build-site.mjs` writes the homepage, `robots.txt`, `sitemap.xml` and **`CNAME`** — on an Actions deploy Pages reads the custom domain out of the artifact, so a build that ships no CNAME drops the domain on the next publish. `SITE_URL` overrides the host for canonicals, sitemap and CNAME.

`mcp/server.mjs` is the **MCP server** — `npx nsds-mcp`, or `npm run mcp`. Six tools (`list_components`, `get_component`, `search_classes`, `list_tokens`, `get_guide`, `get_setup`) reading the real artifacts, with **no dependencies**: it speaks JSON-RPC over stdio directly rather than pulling an SDK into a package whose whole job is to ship CSS. Test it with `echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | node mcp/server.mjs`.

`.claude/skills/namaste-ui/` is the **portable skill bundle** — a self-contained pack for building in this design language from *other* repositories. It is generated by `scripts/build-skill.mjs` from the tokens, the component layer and the `COMPONENTS` array, runs inside `npm run build`, and its `--check` is one of the gates. Never hand-edit it. `docs/PORTABLE-SKILL.md` explains what it is, how to install it elsewhere and why it is separate from the root `SKILL.md` (which is for working *inside* this repo).

`LIVE.md` covers the URLs, the DNS records and troubleshooting. `OBS.md` covers building the documented lesson and live-stream scenes in OBS Studio.
