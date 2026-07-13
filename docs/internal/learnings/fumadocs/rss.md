# Fumadocs RSS Feed

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/guides/rss). Reference for adding RSS/Atom feeds — directly relevant for issues #1 (blog) and #2 (changelog).

---

## 1. Overview

Fumadocs ships a guide for generating **RSS 2.0** feeds (via the `feed` npm package) from any source loader. The pattern:

1. A pure function `getRSS()` that builds the feed XML from `source.getPages()`.
2. A framework-specific route handler that exposes it as an HTTP endpoint.
3. (Optional) `alternates.types` in Next.js `metadata` so browsers auto-discover the feed.

The same pattern can produce **Atom 1.0** (Feed also supports `feed.atom1()`).

---

## 2. Setup

### Install

```bash
pnpm add feed
```

### Create `src/lib/rss.ts`

```ts
import { Feed } from 'feed';
import { source } from '@/lib/source';

const baseUrl = 'https://fumadocs.dev';  // ← adopt this from metadataBase

export function getRSS() {
  const feed = new Feed({
    title: 'Fumadocs Blog',
    id: `${baseUrl}/blog`,
    link: `${baseUrl}/blog`,
    language: 'en',

    image: `${baseUrl}/banner.png`,
    favicon: `${baseUrl}/icon.png`,
    copyright: 'All rights reserved 2025, Fuma Nama',
  });

  for (const page of source.getPages()) {
    feed.addItem({
      id: page.url,
      title: page.data.title,
      description: page.data.description,
      link: `${baseUrl}${page.url}`,
      date: new Date(page.data.lastModified),

      author: [{ name: 'Fuma' }],
    });
  }

  return feed.rss2();
}
```

For **changelog feeds**, the same pattern works — point `source` at the changelog loader instead:

```ts
import { blogSource, changelogSource } from '@/lib/source';

export function getBlogRSS() { /* ... iterate blogSource ... */ }
export function getChangelogRSS() { /* ... iterate changelogSource ... */ }
```

### Expose via a Next.js Route Handler

`src/app/blog/feed.xml/route.ts`:

```ts
import { getBlogRSS } from '@/lib/rss';

export const revalidate = false;

export function GET() {
  return new Response(getBlogRSS(), {
    headers: { 'Content-Type': 'application/rss+xml; charset=utf-8' },
  });
}
```

For changelog: `src/app/changelog/feed.xml/route.ts`.

---

## 3. Next.js Metadata — auto-discovery

Browsers and feed readers discover feeds via `<link rel="alternate">` in the HTML head:

```ts
import type { Metadata } from 'next';

export const metadata: Metadata = {
  alternates: {
    types: {
      'application/rss+xml': [
        {
          title: 'Fumadocs Blog',
          url: 'https://fumadocs.dev/blog/feed.xml',
        },
      ],
    },
  },
};
```

Apply per-route:
- `src/app/blog/layout.tsx` exports the blog feed metadata.
- `src/app/changelog/layout.tsx` exports the changelog feed metadata.

For per-locale feeds (when i18n lands — issue #10), one entry per locale:

```ts
alternates: {
  types: {
    'application/rss+xml': [
      { title: 'Blog (English)', url: `${baseUrl}/blog/feed.xml` },
      { title: 'Blog (Français)', url: `${baseUrl}/fr/blog/feed.xml` },
    ],
  },
},
```

---

## 4. Atom vs RSS 2.0

The `feed` package supports both:

| Format | Method | Audience |
|---|---|---|
| RSS 2.0 | `feed.rss2()` | Most feed readers, legacy compatibility |
| Atom 1.0 | `feed.atom1()` | Modern readers (NetNewsWire, Feedbin), better for i18n |
| JSON Feed 1.1 | `feed.json1()` | Programmatic consumers, JSON-based clients |

For a docs site: **ship RSS 2.0** (default), optionally add Atom 1.0 if you serve international audiences.

To serve both:

```ts
export function GET(request: Request) {
  const accept = request.headers.get('accept') ?? '';
  const body = accept.includes('application/atom+xml')
    ? feed.atom1()
    : feed.rss2();
  const contentType = accept.includes('application/atom+xml')
    ? 'application/atom+xml; charset=utf-8'
    : 'application/rss+xml; charset=utf-8';
  return new Response(body, { headers: { 'Content-Type': contentType } });
}
```

---

## 5. Filtering and sorting

`source.getPages()` returns all pages — for a blog, you want only blog posts:

```ts
export function getBlogRSS() {
  const blogPages = source
    .getPages()
    .filter((page) => page.url.startsWith('/blog/'))
    .sort((a, b) =>
      new Date(b.data.date).getTime() - new Date(a.data.date).getTime()
    );

  // ... add to feed
}
```

For changelog, similar — filter by `/changelog/` prefix, sort by version (descending).

If the blog/changelog is a **separate source** (per issues #1 and #2 design), use `changelogSource.getPages()` directly — no filter needed.

---

## 6. Full content vs excerpt

The default `feed.addItem()` snippet above uses `description` only. For a full-content feed:

```ts
feed.addItem({
  id: page.url,
  title: page.data.title,
  link: `${baseUrl}${page.url}`,
  date: new Date(page.data.date ?? page.data.lastModified),
  content: await page.data.getText('processed'), // or 'raw'
});
```

Tradeoffs:
- **Full content** — better for offline reading, but bigger XML payload (slower to fetch, larger to cache).
- **Excerpt** — leaner, encourages click-throughs to the actual site.

**Recommendation**: excerpts by default (10-15 lines), with the feed XML under ~50KB per item.

---

## 7. Caveats

- **`page.data.lastModified` may be undefined** for pages that don't track modification dates. Fall back to `page.data.date` or build time.
- **Image references inside the feed** must be absolute URLs (use `metadataBase`).
- **XML escaping**: special characters (`<`, `>`, `&`, `"`, `'`) in titles/descriptions are handled by the `feed` package, but custom fields need manual escaping.
- **Rate limits**: a high-traffic feed can hammer your server if cached naively. Use `revalidate = false` for fully static, or `revalidate = 3600` for hourly refresh.
- **Author info**: if pages have multiple authors or no author data, the feed may have empty `<author>` tags. Document the expected `author` frontmatter schema.

---

## 8. Specific considerations for `documentation-template`

### When to add RSS

RSS is **optional** for a docs site. Worth shipping when:
- Blog is enabled (issue #1) — RSS is the blog's killer feature.
- Changelog is enabled (issue #2) — users subscribe to release notifications.
- Docs site is large enough to warrant subscription.

Skip if:
- Only documentation, no blog, no changelog.
- Adoption is internal-only (no public RSS readers).

### Recommended `package.json` scripts

```json
{
  "scripts": {
    "rss:blog": "tsx scripts/build-rss-blog.ts",
    "rss:changelog": "tsx scripts/build-rss-changelog.ts"
  }
}
```

Static-build RSS at build time, commit the XML to the repo (vs dynamic route handler). Pros: zero runtime cost, predictable, can be served as static assets. Cons: must rebuild on content change.

Or use the route handler (as the docs show) for dynamic generation. Both are valid.

### Recommended approach for the template

**Use route handlers (dynamic)** — easier to ship, integrates with the existing `revalidate = false` pattern of `/llms-full.txt` and `/og/docs/...`. Cache via Next.js's automatic static rendering.

When issue #1 lands:
```ts
// src/app/blog/feed.xml/route.ts
import { getBlogRSS } from '@/lib/rss';
export const revalidate = false;
export function GET() {
  return new Response(getBlogRSS(), {
    headers: { 'Content-Type': 'application/rss+xml; charset=utf-8' },
  });
}
```

When issue #2 lands: same pattern for `/changelog/feed.xml`.

### Interactions with other issues

- **#1 blog**: RSS is the natural counterpart. Add to acceptance criteria.
- **#2 changelog**: same — RSS lets users track releases.
- **#4 rich metadata**: `alternates.types` for feed auto-discovery fits naturally in the same `generateMetadata` extension.
- **#10 i18n**: per-locale feeds (already noted in §3).
- **#9 versioning**: per-version feeds (e.g., `/v1/changelog/feed.xml`).

---

## 9. Open questions

- [ ] Does the `feed` package handle MDX components correctly inside `content`? (probably not — would render as raw JSX.)
- [ ] What's the right default cache duration — static (`revalidate = false`) or hourly?
- [ ] Should the feed include the page's last-modified git date, or just frontmatter `date`?
- [ ] Does Fumadocs provide a per-locale feed example for i18n setups?

---

## 10. References

- Official docs: <https://www.fumadocs.dev/docs/guides/rss>
- `feed` package: <https://github.com/jpmonette/feed>
- Sibling issue: #1 (blog support) — RSS is a recommended acceptance criterion
- Sibling issue: #2 (changelog support) — RSS enables release subscriptions
- Sibling issue: #4 (rich metadata) — `alternates.types` for auto-discovery
- Reference learning: `docs/internal/learnings/fumadocs/page-conventions.md` (page metadata)
