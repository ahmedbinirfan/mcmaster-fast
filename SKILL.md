---
name: mcmaster-fast
description: >-
  Apply McMaster-Carr-style techniques that make websites feel instant:
  HTML-first render, app-shell navigation, intent prefetch, layered caching,
  critical CSS, page-specific JS, and layout stability. Use when the user
  wants a site to be extremely fast or responsive, mentions McMaster-Carr /
  mcmaster.com, instant pages, hover prefetch, or asks to implement these
  techniques on another website.
---

# McMaster-fast

Make other sites feel like [mcmaster.com](https://www.mcmaster.com/): first
paint in tens of milliseconds, next page before the URL bar updates.

The stack does not matter. McMaster does this on ASP.NET, YUI, and a
ScriptCombiner. Ordering and request count matter. Framework fashion does not.

## What they actually do

Observed on the live site (2026-09-13) plus the Wes Bos teardown
([klinke.studio notes](https://www.klinke.studio/notes/design/ux/website-speed-and-performance)):

| Layer | What McMaster does | Why it feels instant |
|---|---|---|
| First byte | Server-rendered HTML. Browser paints before JS. FCP ~64ms, TTFB ~19ms on a warm shell. | User sees a page, not a spinner waiting for a bundle. |
| Shell | `#ShellLayout_Cntnr` + `#MainContent`. Header/cart stay mounted. `history.pushState`. | Category click kept the same header node and swapped content in **35ms**. |
| Predict | Hover/intent fetches the next HTML before click. | Click is a cache hit + DOM swap. |
| Repeat visit | Service worker cache-first for GET navigations (not PDFs). Cache held `/` after first load. | Origin HTML is ~200ms (`mc-turn`); SW serves it locally. |
| Waterfall | `dns-prefetch` image hosts. `preload` woff2 fonts, logo, LCP sprites, page JS/CSS, `ShellDynamic.aspx`. | Browser starts fonts/LCP before it discovers them in CSS. |
| CSS | ~32KB common CSS inlined in `<style>` in `<head>`. Rest via a page-specific combiner. | First layout needs no extra CSS request. |
| JS | Page-specific `ScriptCombiner` bundles (`?files=…`). Async. WebParts load after paint. | Catalog page does not download checkout code. |
| Assets | Versioned URLs (`/mv1789156784/`, `?ver=`). `Cache-Control: public,immutable,max-age=31536000`. Image host `images1.mcmaster.com`. Sprites for tile grids. | Year-long cache, no invalidation dance, few requests. |
| Stability | Reserved boxes / sprites. Measured CLS **0**. | No reflow while images arrive. |
| Weight | First-party analytics (`/trk/`). No ads. CSP locked to `*.mcmaster.com`. | The page is the product. |

HTML documents themselves are `Cache-Control: no-cache` and
`x-check-cacheable: NO` at Akamai. Do **not** copy "CDN-cache all HTML".
They cache the **shell in the browser**, and cache **immutable static files**
at the edge.

## Priority ladder

Stop at the first rung that gets the page under ~200ms LCP on a warm cache.
Do not install Workbox, a sprite pipeline, *and* Speculation Rules in one PR.

1. **Ship HTML the browser can paint.** SSR or static HTML. No client-only
   render of the first view. Kill the JS that builds the first screen.
2. **Unblock first paint.** Inline only above-the-fold CSS (or one tiny
   blocking sheet). `preload` the LCP image and woff2 fonts (`crossorigin`).
   `width`/`height` (or CSS aspect-ratio) on every image.
3. **Immutable static files.** Fingerprint or version query. 
   `Cache-Control: public, immutable, max-age=31536000`.
4. **Persistent chrome + content swap.** Shared layout. Next navigation
   replaces one content node and `pushState`s. Do not remount the header.
5. **Intent prefetch.** Speculated next document on hover / viewport.
   Cap it. Never prefetch checkout, search APIs, or authenticated HTML.
6. **Navigation SW (optional).** Cache-first GET navigations for the
   public shell/home. Wipe on deploy. Invalidate from the page, not a
   clever SW. Skip if the site is mostly private.
7. **Page-specific JS.** Route-split. Defer analytics until idle.
   Delete third-party tags that are not the product.
8. **Sprites / image host** only if you have dozens of tiny icons on one
   view. HTTP/2 + a few compressed images is enough below that.

## Agent workflow

When asked to make a site fast this way:

1. Trace the real first-load and one in-app navigation (network + what
   paints). Name the current LCP element and the JS that blocks it.
2. Apply the lowest ladder rung that is missing. One change per pass
   unless the user asked for a sweep.
3. Prefer platform features already in the repo (Next `<Link>`,
   `next/font`, App Router layouts) over new libraries.
4. After a pass, leave **one** check: LCP / FCP numbers, or a tiny
   assert that the shell node is reused across a client navigation.
5. Read [techniques.md](techniques.md) for the exact recipe of the
   rung you are implementing. Read [recipes.md](recipes.md) for
   Next.js / vanilla / existing-SPA mappings.

## Hard rules

- HTML is the product. JS is optional enhancement.
- Prefetch is a hint, not a crawler. Same-origin GET documents only.
- Personalized HTML is `no-store` or SW-only with a short TTL. Never
  put cart/account HTML on a shared CDN cache.
- Do not add a service worker that intercepts every asset. Navigation
  documents only, matching McMaster's `ServiceWorkerJS.aspx`.
- Do not copy YUI, sprites, or ASP.NET. Copy the **order of operations**.
- Do not "modernize" a fast HTML page into a client SPA as the fix.
- Third-party scripts (ads, tag managers, chat) are assumed guilty.
  Keep them off the first paint or cut them.
- Mark shortcuts with `ponytail:` when you skip a lower-value rung.

## Done when

- First visit: HTML paints before any app bundle runs. LCP is a
  server-sent element, not a client-rendered one.
- Next click: chrome stays mounted; only the content region swaps.
- Repeat visit to the home/shell: served locally or from memory cache.
- Fonts and LCP image are in the first 1–2 round trips, not discovered
  after CSS.
- CLS stays near 0 on the pages you touched.
