# Fumadocs Page Conventions

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/page-conventions). Reference for how content files map to URLs, the page tree, and the sidebar.

---

## 1. Overview

Fumadocs generates **page slugs** (URL segments) and **page tree** (sidebar items) from the content directory. The routing is handled by the React framework (Next.js here). Conventions are file-system based:

```
content/docs/
├── index.mdx
└── getting-started.mdx
```

Two files → two pages, two sidebar items. The `loader()` API reads this tree once and exposes it via `source.getPages()` / `source.getPageTree()`.

---

## 2. Page frontmatter

```mdx
---
title: My Page
description: Best document ever
icon: HomeIcon
---

## Learn More
```

Recognized properties:

| Key | Purpose |
|---|---|
| `title` | Page title (also the `<h1>` in Fumadocs UI) |
| `description` | Page description (also the `<meta description>` and OG description) |
| `icon` | Lucide icon name (requires `lucideIconsPlugin` in the loader — already configured in this template) |

For `documentation-template`, the frontmatter schema is locked in `source.config.ts`:

```ts
export const docs = defineDocs({
  dir: 'content/docs',
  docs: { schema: pageSchema, postprocess: { includeProcessedMarkdown: true } },
  meta: { schema: metaSchema },
});
```

To extend, add custom properties to `pageSchema` and use them via `page.data.<yourKey>` in layouts.

---

## 3. Slugs — how files become URLs

| Path (relative to `content/`) | URL slugs | Resulting URL |
|---|---|---|
| `./dir/page.mdx` | `['dir', 'page']` | `/docs/dir/page` |
| `./dir/index.mdx` | `['dir']` | `/docs/dir` |

So `index.mdx` is the URL root of its folder — no `/index` suffix in the URL.

---

## 4. Folder grouping

### Group folder — `(name)`

Wrapping a folder name in parentheses **hides it from slugs** but keeps the grouping:

```
(framework)/
├── index.mdx
└── getting-started.mdx
```

→ `/docs/index` and `/docs/getting-started` (no `framework/` prefix).

Use this for **sidebar organization** without affecting URLs. Example: group "Getting Started" pages together without polluting URLs.

### Root folder — `meta.json` with `"root": true`

```json
{
  "title": "Framework",
  "description": "Framework docs",
  "root": true
}
```

A root folder is rendered as a **Layout Tab** in Fumadocs UI — switching tabs swaps the visible sidebar to that root's pages. Currently hidden folders are not rendered.

Example layout:
```
content/docs/
├── (framework)/     ← root folder A
│   ├── index.mdx
│   └── current-page.mdx
└── (headless)/      ← root folder B
    └── my-page.mdx
```

When viewing `/docs/current-page`, only `framework` items show. The "headless" tab switches to those.

For `documentation-template`, the current setup is single-root. Root folders are a useful pattern for the **i18n issue (#10)** — each locale could be a root.

---

## 5. `meta.json` — sidebar control

Place `meta.json` in any folder to control its sidebar:

```json
{
  "title": "Display Name",
  "icon": "MyIcon",
  "pages": ["index", "getting-started"],
  "defaultOpen": true
}
```

| Key | Purpose |
|---|---|
| `title` | Display name (overrides folder name) |
| `icon` | Lucide icon for the sidebar entry |
| `defaultOpen` | Auto-expand the folder |
| `collapsible` | Whether the folder can collapse (default `true`) |
| `pages` | Explicit ordering of items (see §6) |
| `pagesIndex` | Override the folder index (default = the file named `index`) |

---

## 6. `pages` ordering — full syntax

By default, items sort alphabetically. `pages` lets you control order, add separators, links, and exclusions.

```json
{
  "pages": [
    "components",
    "---My Separator---",
    "...folder",
    "...",
    "!file",
    "!otherFolder",
    "[Vercel](https://vercel.com)",
    "[Triangle][Vercel](https://vercel.com)"
  ]
}
```

| Type | Syntax | Purpose |
|---|---|---|
| Path | `./path/to/page` | Include a page or folder |
| Separator | `---Label---` or `---[Icon]Label---` | Visual break |
| Link | `[Text](url)` / `[Icon][Text](url)` | External/internal link in sidebar |
| External link | `external:[Text](url)` | Same, but opens new tab |
| Rest | `...` | Include remaining pages alphabetically |
| Reversed rest | `z...a` | Same but reversed |
| Extract | `...folder` | Insert all items from a folder inline |
| Except | `!item` | Exclude an item from `...` / `z...a` |

### Constraint: no duplicate URLs

> The same page URL must not appear more than once in the entire page tree. Duplicated page items are not allowed.

This is intentional — Fumadocs locates the active tree item purely from the current pathname. If you violate this rule, the sidebar highlight breaks.

---

## 7. Folder index

By default, a folder's index file (named `index.mdx`) becomes clickable in the sidebar — clicking the folder name navigates to it.

To use a different file as the index:

```json
{
  "pagesIndex": "overview"
}
```

To make the index an external link instead:

```json
{
  "pagesIndex": "[Vercel](https://vercel.com)"
}
```

---

## 8. Icons

Fumadocs doesn't ship an icon library — you provide the resolver. In `documentation-template`, this is done in `src/lib/source.ts`:

```ts
import { lucideIconsPlugin } from 'fumadocs-core/source/lucide-icons';

export const source = loader({
  baseUrl: '/docs',
  source: docs.toFumadocsSource(),
  plugins: [lucideIconsPlugin()],
});
```

This resolves any string in `icon: SomeName` frontmatter to the corresponding `lucide-react` icon.

If the icon name doesn't exist in lucide-react, the page renders without an icon (silent fallback). Use the [lucide.dev](https://lucide.dev) gallery to find valid names.

---

## 9. i18n routing — `dot` vs `dir` parser

The i18n setup from `internationalization.md` accepts a `parser` option in `i18n.ts`:

```ts
import type { I18nConfig } from 'fumadocs-core/i18n';

export const i18n: I18nConfig = {
  parser: 'dot',  // default
  // or
  parser: 'dir',
};
```

### `parser: 'dot'` — suffix on filename

```
content/docs/
├── meta.json
├── meta.cn.json       ← Chinese meta
├── get-started.mdx
└── get-started.cn.mdx ← Chinese page
```

All locales share the same folder structure; locale is encoded in the filename suffix. For the **default locale** (English here), the suffix is optional.

### `parser: 'dir'` — locale-prefixed folders

```
content/docs/
├── (en)/
│   ├── index.mdx
│   └── components.mdx
└── (fr)/
    ├── index.mdx
    └── components.mdx
```

Each locale lives in its own folder. Folder names follow the `(lang)` convention.

### Which to pick

| Use case | Recommended parser |
|---|---|
| Translation always mirrors the source structure 1:1 | `dot` |
| Locales have different structures or content depths | `dir` |
| Many locales (5+) with shared structure | `dot` |
| Few locales (2-3) with shared structure | `dir` (more readable) |

For `documentation-template` initial i18n (#10) with English + French, **`dir` is more readable** and matches the recommendations in the i18n learning doc.

---

## 10. Specific considerations for `documentation-template`

### Current state

```
content/docs/
├── index.mdx      ← title: Hello World
└── test.mdx       ← title: Components
```

No `meta.json` exists. The page tree is alphabetical default. The sidebar shows "Hello World" then "Components".

### Minimum improvement to ship

Create `content/docs/meta.json`:

```json
{
  "title": "Documentation",
  "pages": ["index", "test"]
}
```

This explicitly orders the pages (instead of relying on alphabetical sort) and sets a sidebar title.

### What to add per page

For each MDX file, ensure frontmatter has:

```mdx
---
title: <short, imperative>
description: <one-line, ~150 chars>
icon: <LucideIconName>
---
```

`description` is critical for SEO (issue #4) and OG images.

### Interactions with other issues

- **#3 sitemap**: `source.getPages()` returns slugs; build URLs from them.
- **#4 rich metadata**: page-level OG + Twitter pulls from `title` + `description`.
- **#5 JSON-LD**: `TechArticle` schema reads `title`, `description`, and ideally `date` (which would need to be added to `pageSchema`).
- **#9 versioning**: each version becomes a root folder; the parser stays the same.
- **#10 i18n**: parser choice (`dot` vs `dir`) per §9.

---

## 11. Open questions

- [ ] Can `pageSchema` be extended to include custom properties (e.g. `date`, `author`, `tags`) without breaking existing pages?
- [ ] For nested folder groups, how deep can `(group)/` go before slug resolution gets ambiguous?
- [ ] When mixing root folders and `parser: 'dir'`, what happens — does each locale get its own root tab structure?
- [ ] Does `defaultOpen` cascade to child folders, or apply only to the immediate one?

---

## 12. References

- Official docs: <https://www.fumadocs.dev/docs/page-conventions>
- Page tree API: <https://fumadocs.dev/docs/headless/page-tree>
- Icons: <https://lucide.dev>
- `lucideIconsPlugin`: in `src/lib/source.ts` (already configured)
- Sibling learnings: `internationalization.md`, `markdown.md`
