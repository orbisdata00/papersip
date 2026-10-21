# Papersip site: DevOps guide

How the site is built, published and run. For content and design notes see [README.md](README.md).

## At a glance

| Item | Value |
|---|---|
| Repo | https://github.com/arham993/papersip |
| Branch served | `main`, repository root |
| Hosting | GitHub Pages (static files, no server) |
| Live URL | https://arham993.github.io/papersip/ |
| Stack | React 18, React Router 6, Vite 5, Three.js r128 |
| Node | 18 or newer (built and tested on Node 22) |
| Base path | `/papersip/` (override with `BASE_PATH`) |

## How deployment works

There is no CI. The built site is **committed to the repo root**, and GitHub Pages serves it as is.

```
app/src  ──npm run build──▶  app/dist  ──publish.mjs──▶  repo root  ──git push──▶  GitHub Pages
```

1. `vite build` bundles the client into `app/dist`.
2. `vite build --ssr` builds a server bundle into `app/dist-ssr`.
3. `scripts/prerender.mjs` renders every route to `dist/<route>/index.html`, inlines the CSS and writes `404.html`.
4. `scripts/publish.mjs` deletes the build-owned paths at the repo root and copies `app/dist` over them.

`publish.mjs` only touches the paths in its `OWNED` list: `index.html`, `404.html`, `.nojekyll`, favicons, `brand`, `streams`, `partners`, `assets`, `fonts`, `thumbs`, `work`, and the page folders (`about`, `services`, `clients`, `our-work`, `contact`). `app/`, `README.md`, `DEVOPS.md` and `.git` are never touched.

> **If you add a new top-level folder to `app/public/` or a new page route, add its name to `OWNED` in `app/scripts/publish.mjs`.** Otherwise stale copies stay in the repo root after a rebuild.

## Release checklist

```bash
cd app
npm ci                      # first time, or after package.json changes
npm run publish             # build + pre-render + copy to repo root
cd ..
git status                  # expect changes in app/src and the generated root files
git add -A
git commit -m "Describe the change"
git push origin main
```

GitHub Pages usually finishes updating within 1 to 2 minutes of the push. Check **Settings → Pages** or the **Actions** tab ("pages build and deployment") if it seems stuck. Hard-refresh (Ctrl/Cmd+Shift+R) to bypass the browser cache.

Commit source and generated output together, in the same commit, so the repo root always matches `app/src`.

## Local development

```bash
cd app
npm run dev          # hot-reload dev server (uses the /papersip/ base)
npm run build        # production build only (no copy to root)
npm run preview      # serve app/dist locally
```

To preview the exact published output, serve a folder that contains `papersip/` → `app/dist` (a symlink works) with any static server, e.g. `python3 -m http.server 8766`, then open `http://localhost:8766/papersip/`.

## Custom domain (papersipcompany.com)

1. Build for the domain root: `BASE_PATH=/ npm run publish`.
2. Add a `CNAME` file at the repo root containing the domain. Add it to `app/public/CNAME` and to `OWNED` so rebuilds keep it.
3. In **Settings → Pages**, set the custom domain and enable HTTPS.
4. DNS: `A` records for the apex to GitHub Pages (185.199.108.153, 185.199.109.153, 185.199.110.153, 185.199.111.153), or a `CNAME` for `www` to `arham993.github.io`.

Never mix base paths: a build made with `/papersip/` links break on a domain root, and the reverse.

## Assets and size budget

| Folder (`app/public/`) | Contents |
|---|---|
| `fonts/` | 4 self-hosted WOFF2 files (Latin subset) |
| `brand/` | Logo and footer wordmark, WebP |
| `partners/` | Media-partner logos, WebP |
| `streams/` | Home channel-card photos |
| `thumbs/` | Pre-rendered 3D model stills, WebP (3 sizes each) |
| `work/` | Our Work media: WebP photos (full and `-sm`), H.264 MP4 clips, MP3 radio spots (about 13 MB) |

- Keep images as WebP, about 1400 px wide at most, with a `-sm` (about 640 px) version for grids.
- Video: H.264 + AAC, `-movflags +faststart`, 480 to 540 px high. Audio: MP3, 96 kbps.
- Media loads only on click, so size affects repo weight, not page speed. GitHub warns above 50 MB per file and 1 GB per repo; Pages sites should stay under 1 GB.
- Before changing 3D model colours, regenerate the stills (see `app/scripts/stills.html`).

## Performance and quality checks

Mobile Lighthouse scores sit at 95 to 98. After a significant change, run Lighthouse (mobile) on `/`, `/services/` and `/our-work/` and watch for:

- **Total Blocking Time**: keep Three.js behind the first interaction or the post-load timer. Never import it at the top of a page.
- **CLS**: new images need width/height or an aspect-ratio box.
- **Hydration errors** in the console: the server render and the client render must produce the same markup. Don't read `window` or `Math.random()` during render.

A quick smoke test of the published output: every route returns 200, no console errors, no broken `<img>`, and no `.rv` element stuck at opacity 0 after scrolling.

## Content sources and external dependencies

- All text lives in `app/src/data.js`.
- **Still hosted elsewhere**: client logos and the older OOH, signage and print galleries on `papersipcompany.com/wp-content/uploads/`; some media-partner logos on Wikimedia; TV clips on the Services pages from `cariotauto.in`; radio on Services pages from SoundCloud. Tiles that fail to load hide themselves, but if the WordPress site is retired, copy these images into `app/public/` and update `data.js`.
- Contact form: there is no backend. The brief form builds a `mailto:` link and a WhatsApp link, so there is nothing to host or secure.
- No cookies, analytics or third-party scripts are loaded by default.

## Rollback

Every release is a commit, so rolling back is a revert:

```bash
git revert <bad-commit>     # creates a new commit that undoes it
git push origin main
```

Prefer `git revert` over force-pushing. The generated files are in the same commit, so the site returns to the previous state.

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Site 404s on assets or CSS | Build base path doesn't match where it's served. Rebuild with the right `BASE_PATH`. |
| Old content still showing | Pages is still deploying, or the browser cache. Wait, then hard-refresh. |
| Deep link such as `/our-work/` 404s | The page folder wasn't published. Check `OWNED` and that `dist/our-work/index.html` exists. |
| Deleted file still on the site | It wasn't in `OWNED`, so `publish.mjs` didn't clear it. Add the folder, republish, or remove it by hand. |
| `npm run build` fails on prerender | A page throws during server render (usually `window`/`document` use outside an effect). |
| Videos won't play in a test browser | Headless Chromium lacks H.264. Test in Chrome, Safari or Edge. |
| Images missing locally but fine live | Sandbox or firewall blocks `papersipcompany.com` or Wikimedia. |
