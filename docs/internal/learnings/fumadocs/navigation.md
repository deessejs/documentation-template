# Fumadocs Navigation

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/navigation). Reference for layout-level navigation: header links, sidebar items, and versioning strategies.

---

## 1. Overview

Fumadocs UI's layouts render navigation in two distinct places:

- **Layout links** — top-of-page header links (e.g. "Pricing", "Showcase").
- **Sidebar items** — the page tree (left-side, hierarchical).

These are configured independently.

---

## 2. Layout links

> See <https://fumadocs.dev/docs/ui/layouts/links> for full reference.

Configured via the `links` prop on `HomeLayout` / `DocsLayout`:

```tsx
<DocsLayout
  links={[
    { text: 'Pricing', url: '/pricing', external: false },
    { text: 'Showcase', url: 'https://example.com/showcase', external: true },
  ]}
  tree={source.getPageTree()}
>
  {children}
</DocsLayout>
```

Each link can be:
- `text` — display label
- `url` — destination
- `external` — opens in new tab (`target="_blank"`) and adds `rel="noreferrer noopener"`
- `icon` — optional Lucide icon

### Current state in `documentation-template`

`src/lib/layout.shared.tsx` defines `baseOptions()`:

```tsx
export function baseOptions(): BaseLayoutProps {
  return {
    nav: {
      title: appName,
    },
    githubUrl: `https://github.com/${gitConfig.user}/${gitConfig.repo}`,
  };
}
```

No `links` configured. Layouts (`app/(home)/layout.tsx`, `app/docs/layout.tsx`) spread `baseOptions()` directly.

### When to add links

Useful for linking to:
- Marketing pages on the same site
- Frequently accessed external resources (status page, support)
- Cross-section navigation when you have multiple sections

For `documentation-template`, a minimal `links` example could be:

```tsx
links: [
  { text: 'Blog', url: '/blog', external: false },          // if #1 lands
  { text: 'Changelog', url: '/changelog', external: false }, // if #2 lands
],
```

---

## 3. Sidebar items

> See <https://fumadocs.dev/docs/ui/layouts/docs#sidebar-items> for full reference.

The sidebar is auto-generated from the **page tree**, computed by `source.getPageTree()` (or `source.getPageTree(lang)` with i18n).

```tsx
<DocsLayout tree={source.getPageTree()} {...baseOptions()}>
  {children}
</DocsLayout>
```

To customize the tree beyond what `meta.json` allows, manipulate it post-fetch:

```tsx
const tree = source.getPageTree(lang);

// Example: inject a banner at the top
const bannerItem = {
  type: 'separator',
  name: '---Coming Soon---',
};

const augmentedTree = {
  ...tree,
  children: [bannerItem, ...tree.children],
};

<DocsLayout tree={augmentedTree} {...baseOptions(lang)}>
  {children}
</DocsLayout>
```

Use sparingly — manipulating the tree directly bypasses the `meta.json` flow.

---

## 4. Versioning

The docs cover two patterns:

### Partial versioning — folders

Separate versions into folders within the same docs:

```
content/docs/
├── (v1)/
│   ├── index.mdx
│   └── ...
└── (v2)/
    ├── index.mdx
    └── ...
```

Use `(v1)` and `(v2)` as **root folders** (per `page-conventions.md` §4) to render them as **Layout Tabs** in Fumadocs UI. Users switch between versions via the tab UI.

**Pros**: single deployment, shared infrastructure.
**Cons**: more complex content tree; harder to deprecate old versions.

### Full versioning — git branches

Deploy each version as a separate app, optionally on a subdomain:

```
v2.my-site.com → main branch (or `main` deploy)
v1.my-site.com → v1 branch (or `v1` deploy)
```

**Pros**:
- Old branches untouched when you upgrade dependencies.
- Entire site versioned (not just docs).
- Easier to deprecate — just remove the subdomain.

**Cons**:
- Multiple deployments to maintain.
- Cross-version links require explicit URL rewriting.
- Search and LLM endpoints split per version.

### When to pick

| Scenario | Pattern |
|---|---|
| Active development on current version only | **Partial** (folders) |
| Long-term support of old versions | **Full** (branches) |
| Small docs site, simple versioning | **Partial** |
| Marketing site + docs versioned together | **Full** |

For `documentation-template`, the default should be **partial versioning** (cheaper to adopt) with **documentation in CONTRIBUTING.md explaining how to migrate to full**.

This is the implementation target of **issue #9** (content versioning).

---

## 5. Specific considerations for `documentation-template`

### Current state

- No version navigation.
- No layout links (just the GitHub URL in nav).
- Single-version content tree at `content/docs/`.

### What changes per issue

**#9 versioning**:
- Add root folders for versions: `(v1)/`, `(v2)/`.
- Configure `meta.json` with `"root": true`.
- Add a **Version Banner** component for deprecated versions.

**#1 blog** + **#2 changelog**:
- Add to `baseOptions().links`:
  ```tsx
  links: [
    { text: 'Blog', url: '/blog' },
    { text: 'Changelog', url: '/changelog' },
  ],
  ```

**#10 i18n**:
- `baseOptions` takes `locale`.
- Each locale is a separate root folder (if using `parser: 'dir'`).
- `links` may need translation via the `translations` API.

### Layout tabs vs header links

When you have **both** multiple versions and multiple sections, the UI becomes busy:

```
[Tab: v1] [Tab: v2]     ← version tabs (top)
[Logo] [Home] [Docs] [Blog] [Changelog] [GitHub]   ← nav header
[Sidebar] [Content]
```

Decide explicitly: tabs are for **mutually exclusive contexts** (versions, root sections). Links are for **always-visible destinations**.

### Recommended priority for `documentation-template`

1. Keep `nav` simple: logo + GitHub URL.
2. Add `<links>` only when blog and changelog ship (#1, #2).
3. Add `<Tabs>` (Layout Tabs) when versioning ships (#9).
4. Avoid pre-emptive tabs — adds UI complexity that adopters may not want.

---

## 6. Open questions

- [ ] Does Fumadocs expose the active root folder / version as a `useParams` value for custom UI?
- [ ] Can the version banner component be styled via `fumadocs-ui` theme tokens?
- [ ] For full versioning, how does the GitHub link in `baseOptions` change per version?
- [ ] Does the layout support per-locale navigation hints (e.g. "Switch to French") inline?

---

## 7. References

- Official docs: <https://www.fumadocs.dev/docs/navigation>
- Layout links: <https://fumadocs.dev/docs/ui/layouts/links>
- Sidebar items: <https://fumadocs.dev/docs/ui/layouts/docs#sidebar-items>
- Versioning patterns: see `page-conventions.md` §4 for root folders
- `baseOptions`: `src/lib/layout.shared.tsx`
- Layouts: `src/app/(home)/layout.tsx`, `src/app/docs/layout.tsx`
- Sibling learnings: `page-conventions.md`, `internationalization.md`
