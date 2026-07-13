# Fumadocs Custom Content Source

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/integrations/content/custom). Reference for building a non-MDX content source (CMS, GitHub API, etc.).

---

## 1. Overview

Most Fumadocs projects use **Fumadocs MDX** as the content source (the default in `documentation-template`). But for projects where content lives elsewhere — Sanity, Contentful, Notion API, GitHub repo, etc. — Fumadocs supports **custom content sources** via two layers:

1. **Loader API** (high-level) — file-system-like interface for sources that can be coerced to that shape.
2. **Low-level API** — direct control over `DocsLayout` (page tree) + `DocsPage` (page content).

For `documentation-template`, the default is Fumadocs MDX. This learning is relevant **only for adopters** who want to swap the source for a CMS — it informs future template decisions but doesn't change the current default.

---

## 2. Loader API (recommended approach)

The Loader API expects a `Source` interface:

```ts
export interface Source {
  getPages(language?: string): Page[];
  getPage(slugs?: string[], language?: string): Page | undefined;
  // ... more
}
```

You implement `Source` for your custom backend. The loader then handles:
- Generating static params (`generateStaticParams`)
- Building the page tree (sidebar)
- Filtering by locale (when i18n is active)

For most CMS integrations, you write a small adapter that maps CMS data to the `Source` shape:

```ts
// src/lib/source.ts (custom)
import { loader } from 'fumadocs-core/source';
import { cmsClient } from './cms';

const customSource = {
  async getPages(language) {
    const docs = await cmsClient.getDocuments({ language });
    return docs.map((doc) => ({
      slugs: doc.slug.split('/'),
      url: `/docs/${doc.slug}`,
      data: {
        title: doc.title,
        description: doc.summary,
        // ... MDX or HTML content
      },
    }));
  },

  async getPage(slugs, language) {
    const slug = (slugs ?? []).join('/');
    const doc = await cmsClient.getDocument(slug, { language });
    if (!doc) return undefined;
    return {
      slugs: slugs ?? [],
      url: `/docs/${slug}`,
      data: { /* same as above */ },
    };
  },
};

export const source = loader({
  baseUrl: '/docs',
  source: customSource,
});
```

The rest of the docs site (`DocsLayout`, `DocsPage`, OG images, LLM endpoints) works unchanged — it doesn't care where the content comes from.

---

## 3. Low-level API (full control)

When the Loader API is too constraining (e.g., exotic CMS, dynamic per-request content), drop down to the low-level API:

### Page tree — pass directly to `DocsLayout`

```tsx
import { DocsLayout } from 'fumadocs-ui/layouts/docs';
import type { ReactNode } from 'react';

export default function Layout({ children }: { children: ReactNode }) {
  return (
    <DocsLayout
      tree={{
        // Your hand-built or dynamically-computed tree
        name: 'Documentation',
        children: [
          { type: 'page', name: 'Introduction', url: '/intro' },
          { type: 'separator', name: '— Guides —' },
          { type: 'page', name: 'Quick start', url: '/quick-start' },
        ],
      }}
    >
      {children}
    </DocsLayout>
  );
}
```

The page tree is the data structure that drives **all** navigation elements: sidebar, breadcrumbs, footer prev/next, search results. See the [page tree definitions](https://fumadocs.dev/docs/headless/page-tree) for the full schema.

### Page content — render directly

```tsx
import { DocsPage, DocsBody } from 'fumadocs-ui/layouts/docs/page';
import { getPage } from './my-content-source';
import { notFound } from 'next/navigation';

export default function Page({ params }: { params: { slug?: string[] } }) {
  const page = getPage(params.slug);
  if (!page) notFound();

  return (
    <DocsPage toc={page.tableOfContents}>
      <DocsBody>{page.render()}</DocsBody>
    </DocsPage>
  );
}
```

For Markdown/MDX content, `getTableOfContents` is available as a utility. For HTML, you'd need your own TOC extractor.

---

## 4. Pre-rendering for performance

Critical for static export or any high-traffic deployment:

> When pre-rendering pages from a remote source, **make sure to optimize repeated reads during the build phase**.

```ts
// Bad: rate-limited at scale
export async function generateStaticParams() {
  return await Promise.all(
    slugs.map((slug) => fetch(`https://cms.example.com/docs/${slug}`))
  );
}

// Good: clone locally, then read filesystem
// 1. CI step: git clone / sync CMS content to ./content-cache
// 2. Build step: read from ./content-cache
```

For GitHub-based sources, clone the repo at build start, then read files locally. Avoids rate limits and speeds up builds by 10-100x.

---

## 5. Document search — the hard part

> Fumadocs' built-in search assumes Markdown/MDX. For other content, **bring your own**.

Recommended: use a 3rd-party search engine (Orama Cloud, Algolia) rather than the built-in Orama + MDX pipeline.

Both:
- Index structured documents (CMS records, JSON, HTML).
- Provide their own UI components.
- Support multilingual search.

For `documentation-template`, if an adopter swaps to a CMS-based source, document the search migration in CONTRIBUTING.md as a follow-up task.

---

## 6. Real-world integrations

Fumadocs maintains official integrations for:

| CMS | Package | Notes |
|---|---|---|
| Sanity | `fumadocs-sanity` | Real-time, structured content |
| Payload CMS | community template | Headless, self-hosted |
| Strapi | community plugins | Popular open-source CMS |
| Custom GitHub | DIY (see guide) | Docs-as-code in a separate repo |

The integration pattern is similar: implement the `Source` interface (or use low-level APIs) for your CMS.

---

## 7. Specific considerations for `documentation-template`

### Current state

The template uses **Fumadocs MDX** as the default source. No custom source adapter ships.

### When to consider supporting custom sources

- **If adopters repeatedly ask**: maintainers may add a stub or guide.
- **If a specific CMS is named**: add it as an opt-in integration.
- **For now**: document this learning and let adopters write their own adapter.

### Estimated effort

If/when this becomes a priority:

- **Documentation only** (~30 min): add a guide in `content/docs/guides/custom-content-source.mdx` summarizing this learning.
- **Starter adapter for one CMS** (~1 day): e.g., a minimal `fumadocs-sanity` integration with example pages.
- **Multiple CMS adapters** (~1 week each): not recommended unless there's strong demand.

### Interactions with other issues

- **#10 i18n**: custom sources must implement `getPages(language)` for locale filtering. The loader handles this if you use the high-level API.
- **#11 API reference**: OpenAPI via `staticSource` works alongside a custom source (it's a separate loader entry).
- **#12 link checker**: `next-validate-link` works against MDX files. For a CMS source, link validation needs a CMS-specific implementation.
- **#9 versioning**: each version could have its own source loader (different `getPages` per version).

---

## 8. Open questions

- [ ] Does Fumadocs provide an official adapter template for writing custom sources, or is each one bespoke?
- [ ] How does custom source search compose with the existing `/api/search` route?
- [ ] Can a custom source provide its own `lastModified` for the RSS feed (#2 / #1)?
- [ ] Is there a way to mix MDX and custom sources in a single loader (e.g., MDX for docs, Sanity for blog)?

---

## 9. References

- Official docs: <https://www.fumadocs.dev/docs/integrations/content/custom>
- Loader API: <https://www.fumadocs.dev/docs/headless/source-api>
- Page Tree definitions: <https://fumadocs.dev/docs/headless/page-tree>
- Sanity integration: <https://www.fumadocs.dev/docs/integrations/content/sanity>
- Search API: <https://fumadocs.dev/docs/headless/search>
- Reference learning: `docs/internal/learnings/fumadocs/page-conventions.md`
- Reference learning: `docs/internal/learnings/fumadocs/search-orama.md`
