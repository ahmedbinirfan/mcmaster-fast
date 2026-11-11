---
name: mcmaster-fast
description: >-
  Make websites feel instant: HTML-first render, app-shell navigation,
  intent prefetch, layered caching, critical CSS, page-specific JS, and
  layout stability. Use when the user wants a site to be extremely fast
  or responsive, mentions instant pages, hover prefetch, app shell, or
  asks to implement these techniques on another website.
---

# mcmaster-fast

Make other sites feel instant: first paint in tens of milliseconds, next
page before the URL bar updates.

The stack does not matter. Ordering and request count matter. Framework
fashion does not.

## What this approach does

| Layer | Technique | Why it feels instant |
|---|---|---|
| First byte | Server-rendered HTML. Browser paints before JS. | User sees a page, not a spinner waiting for a bundle. |
| Shell | Persistent chrome + content region. Header/cart stay mounted. `history.pushState`. | Category click keeps the same header node and swaps content in tens of ms. |
| Predict | Hover/intent fetches the next HTML before click. | Click is a cache hit + DOM swap. |
| Repeat visit | Service worker cache-first for GET navigations (not PDFs). | SW serves the shell locally on return visits. |
| Waterfall | `dns-prefetch` image hosts. `preload` woff2 fonts, logo, LCP images, page JS/CSS. | Browser starts fonts/LCP before it discovers them in CSS. |
| CSS | Common CSS inlined in `<style>` in `<head>`. Rest via a page-specific bundle. | First layout needs no extra CSS request. |
| JS | Page-specific bundles. Async. Deferred widgets load after paint. | Catalog page does not download checkout code. |
| Assets | Versioned URLs. `Cache-Control: public,immutable,max-age=31536000`. Cookie-less image host. Sprites for tile grids when needed. | Year-long cache, no invalidation dance, few requests. |
| Stability | Reserved boxes / sprites. Target CLS **0**. | No reflow while images arrive. |
| Weight | First-party analytics only. No ads. Tight CSP. | The page is the product. |

HTML documents themselves are `Cache-Control: no-cache` (revalidate at
the edge). Do **not** copy "CDN-cache all HTML". Cache the **shell in
the browser**, and cache **immutable static files** at the edge.

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
  documents only.
- Do not copy a legacy stack. Copy the **order of operations**.
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
