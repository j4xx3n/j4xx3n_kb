# How This Site Is Built & How To Update It

This document explains how the Knowledge Base Directory website (`j4xx3n.com`)
is built, hosted, and maintained, and gives step-by-step instructions for the
common tasks: adding a page, editing content, changing the navigation, and
adjusting the look.

---

## 1. The Short Version

- The site is **Docsify** — a documentation generator that does **all rendering
  in the browser**. There is **no build step**, no `npm install`, no compiled
  output, and no `dist/` folder.
- The repository *is* the website. The Markdown files you see in the repo are
  the pages, served as-is.
- It is hosted on **GitHub Pages**.
- To publish a change: **edit a Markdown file, commit, and push.** GitHub Pages
  redeploys automatically within a minute or two.

If you only ever do one thing, it's this:

```bash
# edit a .md file, then:
git add -A
git commit -m "update content"
git push
```

---

## 2. What Each File Does

```
j4xx3n_com/
├── index.html        ← the entire app: Docsify config + all CSS + JS
├── _sidebar.md       ← the navigation menu (left sidebar)
├── README.md         ← the home page content
├── .nojekyll         ← tells GitHub Pages NOT to run Jekyll (required)
├── bg-tile.jpg       ← tiled background image
├── portswigger/      ← lab-guide content pages (Markdown)
│   ├── client/
│   ├── server/
│   └── advanced/
├── bug-bounty/       ← bug-bounty checklist content pages (Markdown)
│   ├── recon/
│   └── attack/
└── other/
    └── web-dev/
        └── README.md ← this document
```

### `index.html` — the engine

This single file is the whole application. It contains three things:

1. **CDN script/style tags** (bottom of the file) that load Docsify and its
   plugins. Nothing is installed locally — it's all pulled from
   `cdn.jsdelivr.net` at page load:
   - `docsify@4` — the core renderer
   - `search.min.js` — the sidebar search box
   - `docsify-themeable` — the theming layer (uses the `theme-simple-dark` CSS)
   - `docsify-sidebar-collapse` — collapsible sidebar sections

2. **The `window.$docsify` config object** (around line 259). Key settings:
   - `name` — title shown at the top of the sidebar
   - `repo` — the GitHub link shown in the corner/sidebar
   - `loadSidebar: true` — load navigation from `_sidebar.md`
   - `subMaxLevel: 2` — auto-generate a page table of contents down to `##`
   - `search` — search box behavior
   - `plugins` — a small inline plugin that tags `<body>` with `.is-home` on the
     home page so the heading floats without a card

3. **All the custom CSS** (the big `<style>` block) and a bit of **vanilla JS**
   for the draggable sidebar resizer. The visual identity (teal/neon theme,
   frosted-glass panels, background image, buttons, code styling) lives here.

> **Note:** `repo:` in `index.html` currently points to
> `https://github.com/j4xx3n/j4xx3n_kb`, but the actual repository is
> `j4xx3n_com`. Update that line if you want the corner link to be correct.

### `_sidebar.md` — the navigation

A plain Markdown nested list. Every link in the left menu is defined here. This
is the file you edit most often after content. See section 4.

### `README.md` — the home page

Docsify renders `README.md` as the landing page (`/`). It mixes Markdown with
inline HTML (the `.kb-card` welcome box and `.kb-buttons`, both styled in
`index.html`).

### `.nojekyll` — important, don't delete

GitHub Pages runs Jekyll by default, which **ignores files/folders starting with
an underscore** — that would break `_sidebar.md`. The empty `.nojekyll` file
disables Jekyll so Docsify works. Leave it in place.

### Content folders (`portswigger/`, `bug-bounty/`)

Just Markdown files organized in folders. The folder structure is for your own
organization; what actually appears in the menu is whatever you link from
`_sidebar.md`. A `.md` file that exists but isn't linked in the sidebar is still
reachable by URL, just not listed.

---

## 3. Editing or Adding a Content Page

### Edit an existing page
1. Open the relevant `.md` file (e.g. `portswigger/client/xss.md`).
2. Edit the Markdown.
3. Commit and push.

### Add a new page
1. Create a new `.md` file in the appropriate folder, e.g.
   `portswigger/server/newtopic.md`.
2. Write standard Markdown. Conventions used across the existing pages:
   - Start sections with `##` headings (the home page `#` is reserved for page
     titles). `subMaxLevel: 2` means your `##` headings auto-populate the
     right-hand page TOC.
   - Link headings to the source lab/reference where relevant, e.g.
     `## [Reflected XSS](https://portswigger.net/...)`.
   - Use fenced code blocks with a language hint for syntax highlighting:
     ` ```html `, ` ```bash `, etc.
3. **Add a link to it in `_sidebar.md`** (see next section) so it shows in the
   menu.
4. Commit and push.

---

## 4. Changing the Navigation (`_sidebar.md`)

The sidebar is a nested bullet list. Indentation (2 spaces per level) creates
the hierarchy, and the `docsify-sidebar-collapse` plugin makes bold parent items
collapsible.

Patterns in use:
- **Bold text without a link** = a section header/group, e.g.
  `* **Server Side Labs**`.
- **`* **---**`** = a thin divider line under a header (purely cosmetic).
- **`* [Label](./path/to/page.md)`** = an actual navigable link. Always use a
  leading `./` and the real file path/extension.

Example — adding a new page under "Server Side Labs":

```markdown
  * **Server Side Labs**
    * **---**
    * [API Testing](./portswigger/server/api.md)
    * [New Topic](./portswigger/server/newtopic.md)   ← add a line like this
```

Keep links alphabetized within a group to match the existing style (not
required, just the current convention).

---

## 5. Changing the Look / Theme

All styling is in the `<style>` block of `index.html`. Useful entry points:

- **Accent colors** — the CSS variables in `:root` (top of the style block):
  - `--theme-color` (`#00b4d8`, teal) — primary accent, buttons, repo link
  - `--neon` (`#00e5ff`) — hover glow color
  - `--panel-bg`, `--card-bg` — the frosted-glass surface backgrounds
- **Sidebar width** — `--kb-sidebar-w` (default `17rem`). Users can also drag the
  handle on the sidebar edge; that width is saved in `localStorage`.
- **Content width** — `--content-max-width` (`60rem`).
- **Background image** — `body { background-image: url('bg-tile.jpg') }`. The
  image is a pre-mirrored vertical tile so it repeats seamlessly. To change it,
  replace `bg-tile.jpg` (keep the same name, or update the CSS).
- **Reading card / home card** — `.markdown-section` and `.kb-card`.
- **Code block styling** — the `code` / `pre` rules near the bottom.

To swap the base theme entirely, change the `theme-simple-dark.css` link or the
`docsify-themeable` version in the CDN tags.

---

## 6. Previewing Locally

Because everything is client-side, you can preview by serving the folder over a
local HTTP server (opening `index.html` directly via `file://` will not work —
Docsify needs HTTP to fetch the Markdown files).

Any of these work from the repo root:

```bash
# Python (usually preinstalled)
python3 -m http.server 3000

# Node, if you prefer
npx serve .
# or the official Docsify CLI, which adds live-reload:
npx docsify-cli serve .
```

Then open <http://localhost:3000>. Edit a file, refresh the browser to see
changes (the Docsify CLI auto-reloads).

---

## 7. Deploying / Publishing

There is **nothing to build or deploy manually.** GitHub Pages serves the repo
directly:

1. Commit your changes.
2. Push to the branch GitHub Pages is configured to serve (currently `main`).
3. Wait ~1–2 minutes; GitHub Pages rebuilds and serves the new files.

```bash
git add -A
git commit -m "describe your change"
git push
```

### GitHub Pages settings (one-time, already configured)
In the repo on GitHub: **Settings → Pages**
- **Source:** Deploy from a branch
- **Branch:** `main` / `root` (`/`)

The custom domain (`j4xx3n.com`) is configured there as well. If a `CNAME` file
is ever needed for the custom domain, it lives in the repo root.

---

## 8. Quick Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| A page shows "404" inside the app | The link in `_sidebar.md` doesn't match the real file path/extension. |
| Sidebar item missing | You added the `.md` file but didn't add a link line in `_sidebar.md`. |
| Underscore files (`_sidebar.md`) stop loading after a deploy | The `.nojekyll` file was removed — restore it. |
| Local preview is blank / "Loading..." forever | You opened `index.html` via `file://`. Serve over HTTP instead (section 6). |
| Styles/scripts not loading | A CDN URL/version in `index.html` is wrong or the CDN is unreachable. |
| Search box returns nothing | Content was just added; Docsify indexes on load — hard-refresh the page. |

---

## 9. Summary Cheat Sheet

- **Add a page:** create `.md` → link it in `_sidebar.md` → commit & push.
- **Edit a page:** edit `.md` → commit & push.
- **Change menu:** edit `_sidebar.md`.
- **Change home page:** edit `README.md`.
- **Change theme/colors/layout:** edit the `<style>` block in `index.html`.
- **Change Docsify behavior:** edit `window.$docsify` in `index.html`.
- **Never delete:** `.nojekyll`.
- **No build step. The repo is the website.**
</content>
</invoke>
