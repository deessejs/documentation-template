# Fumadocs Twoslash

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/markdown/twoslash). Adds TypeScript hover annotations to code blocks via the Twoslash integration of Shiki.

---

## 1. Overview

**Twoslash** turns TypeScript code blocks into interactive annotations:

- Hover over a variable → see its inferred type.
- Hover over a function → see its signature and JSDoc.
- Hover over an error → see the error message inline.
- Compiler errors render in-place.

This is the killer feature for docs that document a TypeScript library (which `documentation-template` adopters will do — see issue #11).

Example output:

```ts
type Player = {
  /** @default 'user' */
  name: string;
};

const player: Player = { name: 'Hello World' };
//    ^?
```

→ Hovering `player` shows `Player` type with the `name` field's JSDoc expanded.

---

## 2. Setup

### Install

```bash
pnpm add fumadocs-twoslash twoslash
```

### Configure Next.js to externalize typescript + twoslash

`next.config.mjs`:

```js
const config = {
  reactStrictMode: true,
  serverExternalPackages: ['typescript', 'twoslash'],
};

export default withMDX(config);
```

Without this, Next.js tries to bundle the TypeScript compiler and Twoslash — both are huge and need to run at build time only.

### Wire the transformer into MDX

`source.config.ts`:

```ts
import { defineConfig } from 'fumadocs-mdx/config';
import { transformerTwoslash } from 'fumadocs-twoslash';
import { rehypeCodeDefaultOptions } from 'fumadocs-core/mdx-plugins';

export default defineConfig({
  mdxOptions: {
    rehypeCodeOptions: {
      themes: {
        light: 'github-light',
        dark: 'github-dark',
      },
      transformers: [
        ...(rehypeCodeDefaultOptions.transformers ?? []),
        transformerTwoslash(),
      ],
      // Shiki can't lazy-load languages for Twoslash popups — declare upfront
      langs: ['js', 'jsx', 'ts', 'tsx'],
    },
  },
});
```

### Import the CSS

`src/app/global.css`:

```css
@import 'tailwindcss';
@import 'fumadocs-ui/css/neutral.css';
@import 'fumadocs-ui/css/preset.css';
@import 'fumadocs-twoslash/twoslash.css'; /* ← NEW */
```

### Wire the MDX UI components

`src/components/mdx.tsx`:

```tsx
import * as Twoslash from 'fumadocs-twoslash/ui';
import defaultMdxComponents from 'fumadocs-ui/mdx';
import type { MDXComponents } from 'mdx/types';

export function getMDXComponents(components?: MDXComponents) {
  return {
    ...defaultComponents,
    ...Twoslash,
    ...components,
  } satisfies MDXComponents;
}
```

Without this, the Twoslash popups render as raw HTML with no styling.

### Usage in MDX

Add `twoslash` to the code fence meta string:

````md
```ts twoslash
const a: string = 'Hello World';
console.log(a); // [!code highlight]
```
````

That's it. Hover annotations are live.

---

## 3. Twoslash annotations

These are **inline comments** the Twoslash engine recognizes. Common ones:

### `// ^?` — show inferred type

```ts twoslash
const player = { name: 'Alice', score: 42 };
//    ^?
```

### `// ---cut---` — hide code above from output

```ts twoslash
type Player = { name: string };
// ---cut---
const player: Player = { name: 'Alice' };
//    ^?
```

Only the part after `---cut---` renders. Useful for "look at this specific call" examples.

### `// @noErrors` — suppress error annotations

```ts twoslash
// @noErrors
console.g; // intentional typo
```

### `// @errors: 2588` — assert specific error code

```ts twoslash
// @errors: 2588
const a: number = 'not a number'; // Type 'string' is not assignable to type 'number'
```

Asserts that a specific TypeScript error should appear. If the error code is wrong, the doc build fails.

### `// @target: esnext` and other compiler flags

```ts twoslash
// @target: esnext
const x = 1n; // BigInt literal, valid in esnext only
```

Other common flags: `@strict`, `@noImplicitAny`, `@target`, `@module`, `@lib`, `@jsx`.

---

## 4. File-system cache (optional)

Twoslash invokes the TypeScript compiler per code block. For larger docs, this gets slow. Enable filesystem caching:

`source.config.ts`:

```ts twoslash title="source.config.ts"
import { transformerTwoslash } from 'fumadocs-twoslash';
import { createFileSystemTypesCache } from 'fumadocs-twoslash/cache-fs';

transformerTwoslash({
  typesCache: createFileSystemTypesCache(),
});
```

The cache is keyed by file path + content hash. Subsequent builds skip the TypeScript compiler if the code is unchanged.

The cache directory must be:
- Excluded from git (or kept if you want reproducible builds across clones).
- Listed in `.gitignore`.

---

## 5. Specific considerations for `documentation-template`

### Current state

- **Not configured.** Twoslash is opt-in.
- Code blocks render with plain syntax highlighting (no type info).
- The current `source.config.ts` doesn't pass any `rehypeCodeOptions.transformers`.

### Minimum viable setup

1. `pnpm add fumadocs-twoslash twoslash`
2. Update `next.config.mjs` with `serverExternalPackages`.
3. Update `source.config.ts` to add the transformer.
4. Add `@import 'fumadocs-twoslash/twoslash.css';` to `global.css`.
5. Update `src/components/mdx.tsx` to spread `*Twoslash` into the components.

Estimated effort: ~15 minutes for a contributor familiar with the repo.

### When to enable

Enable when the template starts shipping TS code examples (issue #11 territory — once API reference is generated). Until then, plain highlighting suffices.

### Interactions with other issues

- **#8 (shadcn)**: shadcn example code blocks benefit massively from Twoslash — adopters copy components and want type hints.
- **#11 (API reference)**: TypeDoc-generated MDX includes TS snippets. Twoslash makes those snippets hover-interactive.
- **#4 (rich metadata)**: doesn't affect Twoslash directly.
- **#5 (JSON-LD)**: `TechArticle` schema can mention `programmingLanguage: "TypeScript"` per page if Twoslash is the dominant content type.

### Caveats

- **Bundle size**: Twoslash adds ~2 MB to the dev-time MDX processing (not to the production bundle).
- **Build time**: docs with hundreds of TS snippets → 30-60s extra build time without cache, ~5s with cache.
- **Editor compatibility**: VS Code with the `Twoslash Query Comments` extension (or `Shiki` extension) shows the same annotations in preview, making authoring WYSIWYG.

---

## 6. Open questions

- [ ] Does the Twoslash UI need extra Tailwind config to render popups correctly?
- [ ] How does Twoslash interact with shadcn components if a snippet renders a `<Button>`?
- [ ] Should the cache directory be gitignored or committed (CI builds)?
- [ ] Does the Twoslash transformer work with the i18n setup (#10) per locale?

---

## 7. References

- Official docs: <https://www.fumadocs.dev/docs/markdown/twoslash>
- Twoslash notation reference: <https://shiki.style/packages/twoslash>
- `fumadocs-twoslash` package: <https://github.com/fuma-nama/fumadocs/tree/main/packages/twoslash>
- VS Code extension for preview: search "Shiki" in the marketplace
- Sibling learnings: `markdown.md` (general MDX features)
