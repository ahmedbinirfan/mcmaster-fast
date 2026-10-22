# Recipes

Map each McMaster rung onto the stack you are actually in. Use what
the repo already has. Do not add a library that duplicates it.

## Next.js (App Router)

| McMaster | Next.js |
|---|---|
| Server HTML | Server Component page / `generateStaticParams`. Do not put the LCP tree in a Client Component. |
| Shell + swap | `app/layout.tsx` is chrome. `children` is `#content`. `<Link>` does the swap. |
| Intent prefetch | `<Link>` prefetches in-viewport by default. For hover-like: Speculation Rules in the root layout, or `prefetch={true}` on likely next hops. |
| Fonts | `next/font` (self-host, preload built in). Do not load fonts.net / Google CSS. |
| LCP image | `next/image` with `priority` on the one LCP image. `sizes` set. |
| Critical CSS | Keep global CSS tiny. Colocate route CSS. Avoid a huge `globals.css` of unused utilities on every page. |
| Page JS | Default Server Components. `'use client'` only on the interactive island. |
| Immutable assets | `/_next/static` is already hashed. Do not disable it. |
| SW | Usually skip. Add only for a public marketing/catalog shell. |

```tsx
// app/layout.tsx — chrome lives here and must not remount
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <head>
        <script
          type="speculationrules"
          dangerouslySetInnerHTML={{ __html: JSON.stringify({
            prefetch: [{ where: { href_matches: "/products/*" }, eagerness: "moderate" }]
          }) }}
        />
      </head>
      <body>
        <Header />
        <main id="content">{children}</main>
      </body>
    </html>
  )
}
```

`loading.tsx` is a last resort for slow *data*, not a substitute for
fast HTML. If the page can be static or cached, skip the skeleton.

Cache: use the project's existing `fetch` cache / `'use cache'` /
route segment config. Public catalog pages can be static. Cart cannot.

## Vanilla / server templates (Rails, Django, PHP, ASP.NET, Go)

This is McMaster's world. Stay there.

1. Template renders the full page. Shared layout partial = shell.
2. Add the click interceptor + `pushState` from [techniques.md](techniques.md)
   as one `nav.js` module. Progressive enhancement.
3. Speculation Rules snippet in the layout.
4. Inline the 1–2 KB–30 KB of chrome CSS you already have. Do not
   invent a critical-CSS toolchain for a 15 KB sheet — just inline it.
5. Register the navigation-only SW if home/category HTML is public
   and repeat visits matter.

## Existing client SPA (CRA, Vite+React, Vue)

Do not "add prefetch" on top of a blank `#root`. That is the opposite.

1. Move the first route to SSR (meta-framework) or prerender the
   landing HTML. If that is out of scope, say so and only fix
   navigation + asset caching.
2. Split the route the user is on. Dynamic-import everything else.
3. Persist layout: React Router layout routes / Vue layout — do not
   remount the header on every navigation.
4. `rel=modulepreload` the current route's file; not the world.
5. Drop the tag manager from `index.html` if it is in `<head>`.

## Audit an existing site

In DevTools, first load + one click:

- **Document TTFB / FCP / LCP.** If LCP waits on JS, rung 1 is missing.
- **Does the header remount?** `$0` on the header, click a link, see
  if the node identity changes. If it does, rung 4 is missing.
- **Does hover create a document request?** If not, rung 5 is missing.
- **HTML `Cache-Control`.** `no-store` on a public catalog is leaving
  repeat visits on the table (SW or revalidate). `immutable` on HTML
  with a cart cookie is a bug.
- **Blocking `<script>` / `<link rel=stylesheet>` in `<head>`.** Each
  one is a first-paint tax. Inline or defer.
- **Third-party hosts before FCP.** Guilty until proven otherwise.
- **Image without dimensions.** Instant CLS.

McMaster's own numbers on a warm desktop load of `/`: TTFB ~19ms
(SW), FCP ~64ms, load ~413ms, CLS 0, ~78KB HTML decoded. Use them
as a direction, not a SLA.

## What not to copy

- YUI history iframe, jQuery, 800KB of page JS. They win *despite* this
  because HTML is already on screen.
- `viewport width=1024` desktop lock. Keep a real responsive viewport.
- Caching HTML at the CDN when `Set-Cookie` / A/B / geo change the
  body. They marked HTML `x-check-cacheable: NO`.
- A general-purpose Workbox runtime that caches every API.
- Hover-prefetch of the entire sitemap.

## One-file check (vanilla)

If you add shell navigation, leave a check that fails if the header
node is replaced:

```js
// scripts/check-shell.mjs — run after a headless click, or assert in a demo page
const header = document.getElementById('chrome')
await go('/other')
console.assert(document.getElementById('chrome') === header, 'shell remounted')
```
