# NSDS — live site

Everything about where this design system is published, what is on it, how it
gets there, and how to fix it when it breaks.

---

## 1. The URLs

| What | URL |
| --- | --- |
| **Homepage** | **<https://sfdc.imswarnil.com/>** |
| Styleguide | <https://sfdc.imswarnil.com/preview/index.html> |
| Sitemap | <https://sfdc.imswarnil.com/sitemap.xml> |
| GitHub Pages origin | <https://imswarnil.github.io/NSDS-Design-System/> |
| Repository | <https://github.com/imswarnil/NSDS-Design-System> |
| Issues | <https://github.com/imswarnil/NSDS-Design-System/issues> |
| CI / deploy runs | <https://github.com/imswarnil/NSDS-Design-System/actions> |
| Clone (HTTPS) | `git clone https://github.com/imswarnil/NSDS-Design-System.git` |
| Clone (SSH) | `git clone git@github.com:imswarnil/NSDS-Design-System.git` |

The site is **NSDS**, the repository slug is `NSDS-Design-System`, and the npm
package is still `@namaste-salesforce/design-system` — that last one is the
name two products import by, and renaming it breaks both for nothing.

### The custom domain

`sfdc.imswarnil.com` is a **repo-level** custom domain, which means the site is
served at the **domain root** (`/`), not under a repo subpath. Two things pin
it, and both are required:

1. **DNS** — one record at the `imswarnil.com` provider:

   | Type | Name | Value | Proxy |
   | --- | --- | --- | --- |
   | `CNAME` | `nsds` | `imswarnil.github.io` | DNS-only (grey cloud on Cloudflare) |

   Proxying it breaks GitHub's certificate issuance — leave it unproxied at
   least until the cert is issued.

2. **`_site/CNAME`** — written by `scripts/build-site.mjs` on every build. On an
   *Actions* deploy GitHub reads the custom domain out of the uploaded
   artifact; a deploy that ships no `CNAME` silently drops the domain back to
   `imswarnil.github.io` on the next publish. Setting it in Settings → Pages
   alone is not enough. Override for another host with `SITE_URL=…`.

Then: Settings → Pages → Custom domain → `sfdc.imswarnil.com`, and tick
**Enforce HTTPS** once the certificate is issued (can take up to an hour).

**Account-level domain.** The account `imswarnil` also has an account-level
domain (`dev.imswarnil.com`) that applies to every Pages site without one of
its own. The repo-level domain set here takes precedence, so this site no
longer answers on `dev.imswarnil.com/NS-Design-System/` — update any bookmark
you own.

**Paths stay relative.** Even though the site is now at a domain root, every
asset reference in the built site is *relative* and must stay that way: a card
opened directly, a branch preview, or a move to a subpath all keep working.
Never introduce a root-absolute `/assets/...` path in a card, the preview
generator, or a CSS `url()`.

Deep links are shareable: the styleguide is multi-page (one URL per foundation
section, component and doc), e.g.
`https://sfdc.imswarnil.com/preview/color.html`.

---

## 2. What is on the live site

`gulp site` stages a fully self-contained static bundle (~15 MB — 182
styleguide pages, 56 specimen cards, the CSS closure and self-hosted fonts)
into `_site/`. Those three numbers are printed by the build itself, so read
them off `gulp site` rather than trusting this line. It is **not** the repo — `node_modules/`, `docs/`, `scripts/`,
and the git history are all excluded. What ships:

| Path on the live site | Contents |
| --- | --- |
| `/` | `index.html` — the homepage: what NSDS is, the numbers, and a linked index of all 165 generated pages |
| `/robots.txt` | crawl policy + the sitemap pointer |
| `/sitemap.xml` | the homepage and every generated page |
| `/CNAME` | `sfdc.imswarnil.com` — what actually pins the custom domain |
| `/preview/` | the generated multi-page styleguide (home, one page per foundation, one per specimen group) |
| `/dist/` | the flat CSS bundle `nsds.css` + `.min.css` the preview links |
| `/styles.css` | the entry stylesheet the specimen cards link as `../styles.css` |
| `/tokens/` | authored token CSS + generated `tokens.json` / `tokens.js` / `tailwind.css` |
| `/components/css/` | the portable `.ns-*` component layer both products render |
| `/fonts/` | Figtree — variable woff2s (normal + italic, latin + ext) + the OFL licence |
| `/assets/` | icon fonts, logo, images, `theme-init` |
| `/icons/`, `/patterns/`, `/templates/` | specimen source directories |
| `/**/*.card.html` | every specimen, at its real repo-relative path (the preview iframes them) |
| `/.nojekyll` | tells Pages to serve `_`-prefixed paths verbatim, no Jekyll pass |

### Why the homepage is a page and not a redirect

`/` used to be a meta-refresh into `preview/index.html`. A redirect is not a
page: a crawler follows it, indexes the target, and the domain itself has no
entry in any index — `sfdc.imswarnil.com` would be a URL nothing describes.

So the root is a real document with its own content, and — the part that
matters — a complete linked index of all 165 generated pages. That list is the
only route a crawler has into the styleguide, whose own navigation it reaches
from nowhere else. Both the list and `sitemap.xml` are generated from
`preview/pages.json`, which `build-preview.mjs` writes: a page added to the
generator appears in both without anyone maintaining a second list by hand.

Every styleguide page also carries a `<link rel="canonical">`, a meta
description derived from its lede, and Open Graph tags. The canonical is what
stops a branch deploy or a local copy from competing with the real site in an
index.

The styleguide is generated **from the real artifacts**, so the live site
cannot drift from the system: token tables are read out of
`tokens/tokens.json`, the class index is scraped from `components/css/*.css`,
and each specimen is the actual `.card.html` file in an iframe. The light/dark
switch flips the page and its iframed specimens together, and the choice
persists across pages.

---

## 3. How it gets there

`.github/workflows/ci.yml`, two jobs, in order:

```
push / PR ──▶ check ──▶ (main only) deploy ──▶ GitHub Pages
```

**`check`** — every push and every pull request:

1. `npm ci`
2. `npx gulp build` — regenerate token exports and the CSS bundle
3. `npx gulp check` — the five gates: token-export drift, principle violations
   (raw hex in the component layer), component parsing/self-styling,
   chart-palette colorblind checks, stale `dist/`
4. `git diff --exit-code --stat` — proves no generated file was hand-edited and
   nothing was left uncommitted after a regeneration

**`deploy`** — only on pushes to `main`, and only if `check` passed:

1. `npx gulp site` — build + preview + stage `_site/`
2. `actions/upload-pages-artifact@v3` with `path: _site`
3. `actions/deploy-pages@v4`

A failing check blocks the deploy, so the live site is always a build that
passed every gate.

**Timing.** Roughly 1–2 minutes end to end. GitHub's CDN can take another
minute to purge, so hard-reload (⌘⇧R) before concluding a change didn't ship.

**Publish source.** Repo → Settings → Pages → Source is **"GitHub Actions"**
(not "Deploy from a branch"). There is no `gh-pages` branch and none should be
created — `_site/` is git-ignored and exists only inside the CI runner.

---

## 4. Deploying

### The normal path

```bash
git add -A && git commit -m "…" && git push origin main
```

That is the whole deploy. Nothing else is needed.

### Verify before you push

CI will reject anything the local checks reject, so run them first:

```bash
npm run build     # regenerate token exports + dist/
npm run check     # the five gates CI runs
git status        # must be clean — a dirty tree here fails CI's diff gate
```

### Preview the exact bytes that will be published

```bash
npm run site      # stage _site/ locally (gulp build → preview → site)
npx serve _site   # or: python3 -m http.server -d _site 8080
```

The local server serves `_site/` at the root, exactly as production does. To
stage a build for a different host, set `SITE_URL` — it drives the canonicals,
the sitemap and the `CNAME`:

```bash
SITE_URL=https://staging.example.com npm run site
```

### Watch a deploy

```bash
gh run watch                                    # follow the latest run
gh run list --workflow=ci.yml --limit 5         # recent runs
gh api repos/imswarnil/NSDS-Design-System/pages --jq '.status, .html_url, .cname'
```

### Re-deploy without a code change

```bash
gh workflow run ci.yml --ref main   # (requires a workflow_dispatch trigger)
git commit --allow-empty -m "redeploy" && git push
```

---

## 5. Local development

```bash
npm install
npm run dev        # gulp: build → serve → watch → live reload
```

Opens `http://127.0.0.1:4322/preview/index.html`. The dev server serves the
**repository root**, not a copied build folder, so what you see is what ships.
Save any token, component CSS, specimen card or script and within ~400ms the
tokens, bundle and preview regenerate and every open tab reloads (SSE, no
dependency). A broken save prints the error and keeps watching.

| Command | Does |
| --- | --- |
| `npm run dev` / `gulp` | dev loop: build, serve, watch, live reload |
| `npm run build` / `gulp build` | tokens → CSS bundle → preview page |
| `npm run check` / `gulp check` | the five CI checks |
| `npm run site` / `gulp site` | stage the deployable styleguide into `_site/` |
| `npm run serve` / `gulp serve` | serve an existing build, no watching |

Every gulp task is a thin wrapper over `scripts/*.mjs`, so `node scripts/…`
works with no gulp at all.

`preview/` and `_site/` are git-ignored — both are views of the system, not
part of it, and both are one command away. `dist/` **is** committed on purpose:
the Ghost theme's gulp pipeline consumes the flat bundle directly, and
`npm run check:css` proves it matches source.

---

## 6. Consuming the published system

The live site is documentation. Products consume the package, not the site.

```bash
npm i github:imswarnil/NSDS-Design-System
```

```js
import tokens from "@namaste-salesforce/design-system";           // tokens.js
import "@namaste-salesforce/design-system/styles.css";            // full layer
import "@namaste-salesforce/design-system/dist/nsds.min.css"; // flat bundle
import "@namaste-salesforce/design-system/tokens/tailwind.css";   // Tailwind v4 @theme
```

The three things that must match across both consumers — the 4px spacing base,
the `data-theme` attribute, and the `ns-theme` storage key — are documented and
enforced in **`docs/INTEGRATION.md`**. Read that before wiring up the Ghost
theme or the Next.js LMS.

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Site 404s entirely | Pages source not set to "GitHub Actions", or no successful `deploy` run yet | Settings → Pages → Source: GitHub Actions; then push to `main` |
| Page loads, all CSS missing | a root-absolute `/…` path crept in | make the reference relative; re-run `npm run site` and check locally |
| Domain reverts to `imswarnil.github.io` after a deploy | the artifact shipped no `CNAME` | confirm `build-site.mjs` still writes `_site/CNAME`; it is read from the artifact, not from repo settings |
| Custom domain 404s or shows a cert error right after setup | DNS not propagated, or the record is proxied | `dig sfdc.imswarnil.com CNAME` must return `imswarnil.github.io`; set the record to DNS-only and wait for the cert |
| A page is missing from the sitemap | it isn't in `preview/pages.json` | it is generated from `PAGES` in `build-preview.mjs` — if the page exists there, rebuild |
| Icons render as empty boxes | `dist/` font `url()` paths weren't rebased for the bundle's location | check the woff2 paths inside `dist/nsds.css` and rebuild |
| Specimen iframes are blank | the `.card.html` wasn't copied — `build-site.mjs` only walks `*.card.html` and skips dot-dirs, `node_modules`, `dist`, `preview`, `_site` | confirm the filename ends in `.card.html` and sits outside those dirs |
| `check` fails on `git diff --exit-code` | a generated file was hand-edited, or a regeneration wasn't committed | `npm run build && git add -A && git commit` |
| Deploy skipped | the job only runs on `push` to `main`; PR events never deploy | merge to `main` |
| Old content after a green deploy | CDN cache | hard-reload (⌘⇧R), or check the run actually finished |
| Custom domain shows a cert warning | domain verification lapsed | Settings → Pages → verify domain; the cert re-issues automatically |

---

## 8. Ownership

- **Maintainer:** Swarnil Singhai — <swarnilsinghaicse@gmail.com> — <https://github.com/imswarnil>
- **License:** MIT (see `LICENSE`)
- **Package:** `@namaste-salesforce/design-system` (unchanged — the npm name is
  what two products import by; only the *product* name became NSDS)
- **History:** the repo was renamed `imswarnil/design-system` →
  `imswarnil/NS-Design-System` → `imswarnil/NSDS-Design-System`. GitHub
  permanently redirects every old URL and old git remote, so nothing breaks —
  but update any hardcoded link you own, and your own clone:
  `git remote set-url origin https://github.com/imswarnil/NSDS-Design-System.git`
- **Domain history:** the site moved from `dev.imswarnil.com/NS-Design-System/`
  to `sfdc.imswarnil.com/` (a repo-level custom domain, served at the root).
  The old path is not redirected — GitHub Pages cannot redirect between two
  domains it serves — so fix any link you control.
