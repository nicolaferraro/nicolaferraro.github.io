# Project notes

Personal blog of Nicola Ferraro — **Astro 5** + the **MultiTerm** theme
(`stelcodes/multiterm-astro`), deployed to **GitHub Pages** at
`https://www.nicolaferraro.me`.

Migrated from a Jekyll + Minimal Mistakes site (frozen since 2022). MultiTerm
is a **clone-and-own** template: the theme's source lives directly in this
repo (`src/components`, `src/layouts`, `src/pages`, `src/styles`, …), so there
is no `npm update` for it. "Upgrading the theme" means pulling newer files from
upstream and merging — which means **any edit to a theme source file below can
be silently reverted by an upgrade**. This document is the ledger of those
divergences so they can be re-applied.

---

## ⚠️ Theme source-file edits (re-apply after any theme upgrade)

These touch files owned by the theme. They are **not** covered by
`src/site.config.ts` and will be overwritten if theme files are recopied.

### 1. `src/components/TagsSection.astro` — cap the home tag cloud at 15
The home page tag cloud renders *all* tags. Limited to the 15 largest:
```diff
- tagsGroup.sortCollationsLargest().map((tag) => (
+ tagsGroup.sortCollationsLargest().slice(0, 15).map((tag) => (
```
*Why:* with 45 distinct tags the cloud was unwieldy. `.slice(0, 15)` keeps the
most-used tags only. There is no config option for this in the theme.

### 2. `src/components/Footer.astro` — dynamic copyright year
The theme hardcoded the year as a string literal (`© 2025`). Made it compute
at build time:
```diff
- {siteConfig.author} © 2025
+ {siteConfig.author} © {new Date().getFullYear()}
```
*Why:* the literal never updates. `getFullYear()` is evaluated during the
build, so each deploy reflects the current year.

### 4. `src/components/GiscusLoader.astro` — use giscus's built-in theme
The theme points giscus at a self-hosted CSS URL (`${origin}/giscus/<theme>.css`).
That URL can't be fetched from `localhost`, so comments render in giscus's
default light theme (black text) during local dev. Since the site is locked to
one theme, switched both `data-theme` (in `loadGiscus`) and the `updateTheme`
`postMessage` to giscus's native theme:
```diff
- script.setAttribute('data-theme', `${origin}/giscus/${theme}.css`)
+ script.setAttribute('data-theme', 'catppuccin_mocha')
```
*Why:* giscus ships a native `catppuccin_mocha` theme that matches the site, so
the self-hosted CSS is unnecessary and the localhost fetch problem disappears.

### 3. `astro.config.mjs` — `build.format: 'file'`
Added inside `defineConfig`:
```js
build: { format: 'file' },
```
*Why (important):* the theme emits links **without** a trailing slash
(e.g. `PostPreview.astro` hardcodes `/posts/${post.id}`), but Astro's default
output is `posts/slug/index.html`. GitHub Pages serves that only at
`/posts/slug/` and does **not** redirect the no-slash form, so every internal
post link 404'd. `format: 'file'` emits `posts/slug.html`, which GitHub Pages
serves cleanly at `/posts/slug`, matching the theme's links. Keep this in sync
with `trailingSlashes: false` in `src/site.config.ts` — the two must agree.

---

## Configuration — `src/site.config.ts`

This is the theme's intended customization surface (lower upgrade risk, but
still merge carefully). Choices made:

- **Identity / SEO:** `site` = `https://www.nicolaferraro.me`, title/description/
  author set to Nicola Ferraro; `tags` set to blog keywords.
- **`pageSize: 8`** (was 6).
- **Single theme, no switcher:** `themes.mode: 'single'`, `themes.default:
  'catppuccin-mocha'`, and `themes.include` trimmed to **just**
  `['catppuccin-mocha']` (upstream bundles 59). With `mode: 'single'` the theme
  selector button is not rendered, and only one Shiki theme is bundled.
- **Social links:** trimmed to `github`, `linkedin`, `rss: true`
  (removed mastodon / bluesky / twitter / email).
- **`navLinks`:** GitHub link points to `https://github.com/nicolaferraro`.
- **Giscus comments:** configured against `nicolaferraro/nicolaferraro.github.io`,
  category `Announcements`. Requires Discussions enabled + the Giscus app
  installed on the repo. `giscus.json` (repo root) `origins` is set to the
  custom domain.
- **`trailingSlashes: false`** — must stay consistent with `build.format: 'file'`
  (see theme edit #3). The Search component reads this value and formats
  pagefind URLs accordingly, so the whole site stays on no-trailing-slash URLs.

---

## Content & assets

- **Posts:** `src/content/posts/<slug>.md`. Frontmatter schema (see
  `src/content.config.ts`): `title`, `published` (date), optional `description`,
  `tags` (string array), `coverImage: { src, alt }`, `draft`, `toc`, `series`.
- **Cover images:** `src/assets/covers/` — referenced from frontmatter as
  `../../assets/covers/<file>` (Astro's `image()` loader resolves relative to
  the markdown file; must live under `src/`, not `public/`).
- **In-post images:** `public/images/…`, referenced with absolute `/images/…`
  paths (served as-is, not optimized).
- **Pages / blurbs:** `src/pages/about.md`, `src/content/home.md`,
  `src/content/addendum.md` are personalized (not theme defaults).
- **Avatar:** `src/content/avatar.jpg` is Nicola's photo (square 268×268, also
  used for social-card generation).
- **Favicon:** `public/favicon.svg` replaced with a `</>` dev glyph
  (bg `#1e1e2e`, stroke `#cba6f7` — Catppuccin Mocha base + mauve).
  Referenced by `src/layouts/Layout.astro` as `/favicon.svg` (no edit needed).

---

## Authoring a new post

1. Create `src/content/posts/<slug>.md`. **The filename is the URL slug** — the
   post is served at `/posts/<slug>` (no date in the path, unlike the old Jekyll
   URLs). Use a short, lowercase, hyphenated slug.
2. Add frontmatter (below), write the body in Markdown, preview with
   `npm run dev` → http://localhost:4321/posts/<slug>.
3. Commit and push to `main`; the workflow builds + deploys. Search index and
   RSS update automatically.

### Frontmatter

Minimal new post (**no cover image** — that's the preference for new posts):
```yaml
---
title: 'My Post Title'
published: 2026-06-07
tags: ['Apache Camel', 'Kubernetes']
---
```
Fields (schema: `src/content.config.ts`):
- `title` — **required**, string.
- `published` — **required**, date `YYYY-MM-DD`. Controls ordering and the
  displayed date (it does *not* affect the URL).
- `description` — optional. Used for SEO/RSS and the post preview. **If omitted,
  the first paragraph is used automatically** (~200 chars).
- `tags` — optional string array. Reuse existing tags for consistency (see
  below). The home page shows the 15 most-used.
- `draft` — optional, default `false`. `true` hides the post from production
  builds but still shows it under `npm run dev`.
- `toc` — optional, default `true`. Set `false` to hide the table of contents.
- `series` — optional string; groups related posts into a series.
- `author` — optional; defaults to the site author.
- `coverImage` — supported by the theme, but **omit it on new posts.**

### Tags in use (reuse these; case-sensitive)
Apache Camel, Kubernetes, JBoss Fuse, Openshift, Camel K, Knative, Serverless,
Apache Spark, Kamelets, Spring Boot, Fabric8, Big Data, Cloud Native, Integration,
Microservices, Java, Scala, Docker, DevOps. The `/tags` archive lists the full set.

### Body format & syntax

Plain **GitHub-flavored Markdown** (`.md`). Use `.mdx` only if you need to embed
Astro/JSX components. MultiTerm extensions available:

- **Headings** get auto anchor links; the TOC is built from them.
- **Code blocks** (Expressive Code) — fence with a language, optional `title=`
  and flags (line highlighting, diffs, line numbers, `wrap`):
  ````md
  ```python title="hello.py"
  print("hi")
  ```
  ````
  Docs: https://expressive-code.com/key-features/syntax-highlighting/ . Inline
  code with backticks.
- **Images** — two options:
  - For an image-heavy post, make it a folder: `src/content/posts/<slug>/index.md`
    and drop images alongside, referenced relatively (`![alt](./foo.png)`) so
    Astro optimizes them.
  - For the occasional image, put it in `public/images/` and reference
    `/images/foo.png` (served as-is, not optimized).
  - A title string becomes a caption: `![alt](./foo.png 'My caption')`. Append
    `#pixelated` to the alt text for pixel art.
- **Admonitions:** `:::note` / `:::tip` / `:::important` / `:::caution` /
  `:::warning`, closed with `:::`.
- **GitHub cards:** `::github{repo="apache/camel-k"}` or `::github{user="apache"}`.
- **Character chats** (optional, playful): `:::owl` / `:::unicorn` / `:::duck`
  blocks, optional `{align="right"}`.
- **Math (KaTeX):** inline `$ ... $`, block `$$ ... $$`.
- **Emoji:** GitHub shortcodes (`:rocket:`) or literal emoji.
- **Tables, blockquotes, lists, strikethrough (`~~`), and raw HTML** all work.

---

## Old-URL redirects (Jekyll → Astro)

Old Jekyll permalinks were `/:year/:month/:day/:title/` (with trailing slash);
new URLs are `/posts/:slug`. To preserve inbound links, **static redirect
stubs** were generated at `public/<year>/<month>/<day>/<slug>/index.html` — one
per post, each a `<meta http-equiv="refresh">` + `rel="canonical"` to
`/posts/<slug>`.

These are plain files in `public/`, so they are **independent of Astro's build
format** and serve at the old trailing-slash URLs regardless of theme settings.
If posts are added/renamed, regenerate them from each post's `published` date +
filename slug.

---

## Build & deploy

- `npm run build` → Astro build + `pagefind` index (postbuild). Local preview:
  `npm run dev` (http://localhost:4321).
- **CI/CD:** `.github/workflows/astro.yml` builds and deploys to GitHub Pages.
  Triggers on **`main`** (the repo's default branch).
- **Custom domain:** `public/CNAME` contains `www.nicolaferraro.me` (deploys
  with the artifact).

### Hosting settings that live outside the repo (GitHub / Cloudflare)
- GitHub Pages **build type must be `workflow`** (Settings → Pages → Source =
  "GitHub Actions"), not the legacy "Deploy from a branch" Jekyll builder.
  Set via API: `gh api -X PUT repos/<owner>/<repo>/pages -f build_type=workflow`.
- GitHub Pages **custom domain** must be set (`-f cname=www.nicolaferraro.me`);
  changing the build type can drop it.
- **Cloudflare DNS:** the `www` (and apex) records must be **DNS-only (grey
  cloud)**, CNAME → `nicolaferraro.github.io`. Proxied (orange cloud) hides the
  CNAME behind Cloudflare IPs, which blocks GitHub's domain verification and TLS
  cert provisioning (symptom: "Site not found"). Purge Cloudflare cache after
  changes.

---

## Theme-upgrade checklist

When pulling a newer MultiTerm:
1. Merge via git (track `stelcodes/multiterm-astro` as a remote) so your commits
   replay and only real conflicts surface.
2. Re-verify the three **theme source-file edits** above (tag cloud limit,
   footer year, `build.format`) — these are the ones an upgrade can clobber.
3. Re-check `src/site.config.ts` against any new/renamed config keys.
4. Rebuild and confirm: post links resolve at no-slash URLs, the tag cloud shows
   15, the footer year is current, and a sample old `/YYYY/MM/DD/slug/` redirect
   still works.
