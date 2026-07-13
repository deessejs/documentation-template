# Fumadocs Markdown / MDX

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/markdown). Reference for content authoring patterns: components, code blocks, callouts, and tab groups.

---

## 1. Overview

Fumadocs extends MDX with a set of opinionated components and remark plugins:

- **Components**: `<Cards>`, `<Card>`, `<Callout>`, `<Tabs>`, `<Accordion>`, `<ZoomableImage>`, etc.
- **Code block features**: line numbers, syntax highlighting, line highlights, word highlights, diff, focus, tab groups, npm commands.
- **Heading features**: auto anchors, TOC visibility control, custom anchor IDs.
- **Other**: `<include>` for file inclusion (Fumadocs MDX only), steps for tutorials.

This template already includes the default set via `src/components/mdx.tsx`:

```ts
import defaultMdxComponents from 'fumadocs-ui/mdx';

export function getMDXComponents(components?: MDXComponents) {
  return { ...defaultMdxComponents, ...components } satisfies MDXComponents;
}
```

The catch-all page (`src/app/docs/[[...slug]]/page.tsx`) wires it via `createRelativeLink`:

```tsx
<MDX components={getMDXComponents({ a: createRelativeLink(source, page) })} />
```

---

## 2. Auto links

Internal links use Next.js's `<Link>` component → prefetching + soft navigation. External links auto-add `rel="noreferrer noopener" target="_blank"`.

```mdx
[My Link](https://github.github.com/gfm)
This also works: https://github.github.com/gfm.
```

Bare URLs are auto-linked.

---

## 3. Cards

```mdx
<Cards>
  <Card
    href="https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating"
    title="Fetching, Caching, and Revalidating"
  >
    Learn more about caching in Next.js
  </Card>
  <Card title="href is optional">Learn more about `fetch` in Next.js.</Card>
  <Card icon={<HomeIcon />} href="/" title="Home">
    You can include icons too.
  </Card>
</Cards>
```

### Auto-rendered "Further Reading" cards

Using `getPageTreePeers` from `fumadocs-core/page-tree`:

```tsx
import { getPageTreePeers } from 'fumadocs-core/page-tree';
import { source } from '@/lib/source';

<Cards>
  {getPageTreePeers(source.getPageTree(), '/docs/my-page').map((peer) => (
    <Card key={peer.url} title={peer.name} href={peer.url}>
      {peer.description}
    </Card>
  ))}
</Cards>;
```

→ Renders the other pages in the same folder as cards. Useful for "Related pages" sections.

This is a candidate for **issue #11** (related pages feature).

---

## 4. Callouts

```mdx
<Callout>Hello World</Callout>
<Callout title="Title" type="warn">Hello World</Callout>
<Callout title="Title" type="error">Hello World</Callout>
<Callout title="Title" type="idea">Hello World</Callout>
```

Types: `info` (default), `warn` / `warning`, `error`, `success`, `idea`.

For `documentation-template`, callouts are useful for **migration warnings** in the contributing guide and **deprecation notices** in versioned content.

---

## 5. Headings & TOC

### Auto-anchors

```md
# Hello `World`
```

→ Anchor `#hello-world` (sanitized, lowercase, spaces → dashes).

### Hide from TOC

```md
# Heading [!toc]
```

Heading is rendered but excluded from the right-side TOC.

### TOC-only heading

```md
# Another Heading [toc]
```

Heading is **not rendered** but appears in TOC. Useful for "fake" section markers added programmatically.

### Custom anchor

```md
# heading [#my-heading-id]
```

Override the auto-generated slug.

Combined:

```md
# heading [toc] [#my-heading-id]
```

→ In TOC, with anchor `/page#my-heading-id`.

---

## 6. Code blocks

Default: syntax highlighting via **Rehype Code** (Shiki under the hood). Themes default to GitHub light/dark via `rehypeCodeDefaultOptions`.

### Line numbers

```md
```ts twoslash lineNumbers
const a = 'Hello World';
console.log(a); // [!code highlight]
```
```

Or with a starting offset:

```md
```js lineNumbers=4
function main() {
  console.log('starts from 4');
  return 0;
}
```
```

### Shiki transformers (inline annotations)

```md
```tsx
<div>Hello World</div> // [!code highlight]
<div>Fumadocs</div>      // [!code word:Fumadocs]
console.log('hewwo');     // [!code --]
console.log('hello');     // [!code ++]
return new ResizeObserver // [!code focus]
```
```

Transformers:
- `// [!code highlight]` — highlight a line
- `// [!code word:Foo]` — highlight a word
- `// [!code --]` / `// [!code ++]` — diff style
- `// [!code focus]` — fade out other lines

### Tab groups

```md
```ts tab="Tab 1"
console.log('A');
```

```ts tab="Tab 2"
console.log('B');
```
```

To persist the selected tab across pages:

```md
```ts tab="Tab 1" tab-group="my-custom-group"
console.log('A');
```

```ts tab="Tab 2"
console.log('B');
```
```

### MDX inside tab values

By default, tab values are plain text. To enable MDX:

```ts
// source.config.ts
import { defineConfig } from 'fumadocs-mdx/config';

export default defineConfig({
  mdxOptions: {
    remarkCodeTabOptions: {
      parseMdx: true,
    },
  },
});
```

Then:

```mdx
```ts tab="<Building /> Tab 1"
console.log('A');
```
```

Enables arbitrary JSX in tab labels. Use sparingly — heavy MDX in tabs slows rendering.

---

## 7. NPM commands

Fumadocs ships a `remark-npm` plugin that expands a `npm` code block into a tab group showing install commands for npm/pnpm/yarn/bun.

````md
```npm
npm i next -D
```
````

Renders as a 4-tab group: `npm` / `pnpm` / `yarn` / `bun`.

For `documentation-template`, this is essential — every "Quick start" block should use ` ```npm ` to avoid hardcoding a package manager.

---

## 8. Steps (tutorial-style)

```md
### Installation [step]
### Write Code [step]
### Deploy [step]
```

Renders as numbered cards visually distinct from regular headings. Use for tutorials with discrete phases.

Requires `remark-steps` to be enabled in `source.config.ts`. Verify whether the current template enables it.

---

## 9. Include (file inclusion)

> Available only on Fumadocs MDX (this template uses Fumadocs MDX).

```mdx
<include>./another.mdx</include>
```

Inlines the content of another file at build time. Useful for:
- Sharing snippets across pages (e.g. install commands).
- Splitting large files into smaller logical chunks.

Verify the full syntax via the [Fumadocs include docs](https://fumadocs.dev/docs/markdown/include).

---

## 10. Other components

Beyond what this page covers, Fumadocs UI also provides:

- `<Tabs>` / `<Tab>` — content tab groups (vs code-block tab groups).
- `<Accordion>` / `<Accordions>` — collapsible content sections.
- `<ZoomableImage>` — click-to-zoom images (good for screenshots, diagrams).
- `<Files>` / `<File>` — file-tree visualization in code blocks.

These need to be enabled via the `fumadocs-ui/mdx` imports — already done in this template.

---

## 11. Specific considerations for `documentation-template`

### What's already enabled (via `src/components/mdx.tsx`)

- `<Cards>` / `<Card>`
- `<Callout>`
- Auto links via `createRelativeLink`
- Code blocks with line numbers
- Tab groups in code blocks
- Shiki transformers (`// [!code ...]`)

### What's NOT enabled by default (would need config)

- **Twoslash** (TypeScript hover annotations) — see `markdown-twoslash.md`
- **NPM commands** (`remark-npm`) — verify in `source.config.ts`
- **Steps** (`remark-steps`) — verify
- **MDX in tab groups** — explicit opt-in via `remarkCodeTabOptions.parseMdx`
- **Include** — should already work via Fumadocs MDX

### Recommended starter content updates

The two example MDX files (`index.mdx`, `test.mdx`) currently show minimal examples. They could be enriched to demonstrate:

- **A real `<Callout>`** for "what's next" guidance.
- **A npm install block** with the ` ```npm ` syntax (instead of plain `bash`).
- **An auto-rendered "Further Reading" Cards section** using `getPageTreePeers`.
- **Heading anchors** + `[!toc]` annotations.

### Interactions with other issues

- **#4 rich metadata**: code block tab groups can include MDX → ensure metadata generation doesn't break on tab content.
- **#5 JSON-LD**: each page becomes a `TechArticle` → code block features don't affect schema, but `description` field becomes critical.
- **#11 API reference**: generated pages from TypeDoc will use the same MDX features — verify TypeDoc's MDX output is compatible.

---

## 12. Open questions

- [ ] Are `remark-npm` and `remark-steps` already enabled in the current `source.config.ts`?
- [ ] What Shiki transformers are bundled by default? (`rehypeCodeDefaultOptions`)
- [ ] Does the `<include>` tag work across locale directories when i18n (#10) lands?
- [ ] Can `<DynamicLink>` from the i18n doc be used inside tab values?
- [ ] Are there MDX security implications for letting contributors include arbitrary components?

---

## 13. References

- Official docs: <https://www.fumadocs.dev/docs/markdown>
- Twoslash: <https://www.fumadocs.dev/docs/markdown/twoslash> — see `markdown-twoslash.md`
- Page conventions: <https://www.fumadocs.dev/docs/page-conventions> — see `page-conventions.md`
- `createRelativeLink`: used in `src/app/docs/[[...slug]]/page.tsx`
- Sibling learnings: `markdown-twoslash.md`, `page-conventions.md`, `internationalization.md`
