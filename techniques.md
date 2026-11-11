# Techniques

Each section: the technique, then the smallest portable implementation.
Use the platform feature in [recipes.md](recipes.md) when the repo already has one.

## 1. HTML first

Send a finished document. The browser paints that HTML. App scripts are
`async` / `defer` / `type="module"`. Deferred widgets XHR in after
`DOMContentLoaded`.

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>…</title>
  <!-- critical CSS, preloads: see below -->
</head>
<body>
  <div id="shell">
    <header id="chrome">…logo, search, cart…</header>
    <main id="content"><!-- server HTML for this URL --></main>
  </div>
  <script type="module" src="/app.js"></script>
</body>
</html>
```

Rules:

- First view = HTML from the server or a static file. No empty `#root`.
- App JS is `type="module"` or `defer`/`async`. Never a blocking bundle
  in `<head>` that the first paint needs.
- Progressive enhancement: links work without JS (full navigation).
  JS only makes the next click a swap.

## 2. App shell + content swap

Keep stable IDs for chrome and content (e.g. `#chrome`, `#content`). A
navigation should keep the **same header node** and only change the URL,
title, and content region.

```js
// ponytail: in-memory page cache, Map is enough until you need IDB
const pages = new Map()

export async function go(url, { push = true } = {}) {
  const html = pages.get(url) ?? await (await fetch(url, { headers: { 'X-Partial': '1' } })).text()
  pages.set(url, html)
  const next = new DOMParser().parseFromString(html, 'text/html')
  const incoming = next.getElementById('content')
  const current = document.getElementById('content')
  if (!incoming || !current) return location.assign(url)
  current.replaceWith(incoming)
  document.title = next.title
  if (push) history.pushState({ url }, '', url)
}

document.addEventListener('click', (e) => {
  const a = e.target.closest('a[href]')
  if (!a || a.origin !== location.origin || a.hasAttribute('data-full')) return
  if (e.metaKey || e.ctrlKey || a.target === '_blank') return
  e.preventDefault()
  go(a.href)
})

window.addEventListener('popstate', () => go(location.href, { push: false }))
```

Server option: same full HTML always (parse out `#content`) **or** a
partial endpoint. Full HTML is simpler and prefetch-friendly.

Fetch chrome extras after DCL to inject deferred `<link>` / `<script>`
into `<head>`. That is "chrome is static, extras are late." Do not block
first paint on them.

## 3. Intent prefetch

Hover (or similar intent) starts the HTML request; click paints from
that response, then `pushState`. Prefer real pointer events over
synthetic ones.

**Prefer the platform** (2023+): Speculation Rules.

```html
<script type="speculationrules">
{
  "prefetch": [{
    "where": { "and": [
      { "href_matches": "/*" },
      { "not": { "href_matches": ["/cart*", "/account*", "/api/*", "/logout"] } }
    ]},
    "eagerness": "moderate"
  }]
}
</script>
```

`moderate` ≈ hover-ish. `conservative` for prerender. Do not
`prerender` authenticated pages.

Fallback (no Speculation Rules): prefetch on `pointerenter` with a
~80ms delay, one in-flight, same-origin GET only. Cap at ~4 prefetches
ahead. Feed `pages` in section 2.

```js
let t
let inflight = 0
document.addEventListener('pointerenter', (e) => {
  const a = e.target.closest?.('a[href]')
  if (!a || a.origin !== location.origin || inflight >= 4) return
  t = setTimeout(() => {
    inflight++
    fetch(a.href).then(r => r.text()).then(h => pages.set(a.href, h)).finally(() => { inflight-- })
  }, 80)
}, true)
document.addEventListener('pointerleave', () => clearTimeout(t), true)
```

Never prefetch POST, files, or logout.

## 4. Service worker (navigations only)

Keep the worker small: cache-first GET navigations only.

```js
self.addEventListener('install', (event) => {
  event.waitUntil(caches.keys().then((keys) => Promise.all(keys.map((k) => caches.delete(k)))))
  self.skipWaiting()
})

self.addEventListener('activate', (event) => event.waitUntil(self.clients.claim()))

self.addEventListener('fetch', (event) => {
  const req = event.request
  if (req.mode !== 'navigate' || req.method !== 'GET') return
  if (/\.pdf($|\?)/i.test(req.url)) return
  event.respondWith(caches.match(req).then((hit) => hit || fetch(req)))
})
```

Wipe caches on install; **invalidation lives in the page**. Populate
from the page after a good response (the worker above does not
`cache.put` on miss — page-side fill is the contract):

```js
if ('caches' in window) {
  caches.open('shell-v1').then((c) => c.add('/'))
}
```

Bump the cache name or wipe on deploy. Unregister + delete caches if
the shell endpoint 404s.

Do not cache `/cart`, `/account`, or anything with `Set-Cookie` that
varies the body.

## 5. Kill the discovery waterfall

Example `<head>` (abridged):

```html
<link rel="dns-prefetch" href="//images.example.com">
<link rel="dns-prefetch" href="//cdn.example.com">
<link rel="preload" href="…/logo.svg?ver=…" as="image" type="image/svg+xml">
<link rel="preload" href="/fonts/….woff2?ver=…" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="…/category-sprite.png?ver=…" as="image">
```

Rules:

- `dns-prefetch` (or `preconnect`) only for hosts you hit in the first
  2s (CDN, font host).
- `preload` the LCP image and the 1–2 woff2 files actually used above
  the fold. Font preload **requires** `crossorigin`.
- Do not preload every sprite or every route's JS.

## 6. Critical CSS

One `<style>` in `<head>` (reset + chrome + first screen). Page CSS
arrives via a separate bundle and can also be `rel=preload`.

```html
<head>
  <style>/* reset + header + first screen only */</style>
  <link rel="preload" href="/page.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
  <noscript><link rel="stylesheet" href="/page.css"></noscript>
</head>
```

Keep the inline block small. If it exceeds ~40KB uncompressed you
inlined too much. Extract with the project's existing CSS pipeline;
do not add Critters unless nothing else can do it.

## 7. Page-specific JS and CSS

Each page lists the modules it needs. A homepage does not download a
product-configurator bundle.

Equivalent: route-level splits, not one `main.js`. Dynamic `import()`
for search-as-you-type, CAD viewers, chat. Measure with the coverage
panel — if a file is unused on that route, it should not be requested.

## 8. Immutable versioned assets

| Resource | Cache-Control |
|---|---|
| HTML (`/`, `/products/…`) | `no-cache` (revalidate) |
| Font / logo / hashed JS/CSS | `public,immutable,max-age=31536000` |

Do this:

- Fingerprint hashes (`file.abc123.js`) **or** a deploy prefix.
- HTML: `Cache-Control: no-cache` (revalidate) or short TTL if public
  and identical for all users.
- Never `immutable` on HTML that contains user state.

## 9. Images and sprites

Minimum everywhere:

```html
<img src="widget.webp" width="160" height="160" alt="…">
```

Sprite only when one view has many tiny shared glyphs (icon toolbars,
category grids). Otherwise compressed WebP/AVIF + `width`/`height`.
Put media on a cookie-less host if the HTML origin sets lots of cookies.

## 10. Stay out of the way

- Analytics: `requestIdleCallback` or `type="module" async` after paint.
- Chat widgets: on interaction, not on load.
- A/B tests that block HTML are a regression. Enroll after the page is up.

## 11. Hash fallback (only if you must)

Without `history.pushState`, some stacks used `location.replace('/#' + path)`.
Skip this on any site that does not need legacy browser support. Mentioned
so you do not reimplement it.
