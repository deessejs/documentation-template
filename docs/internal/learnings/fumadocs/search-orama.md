# Fumadocs Search with Orama

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/search/orama). Reference for the default search engine and customization options.

---

## 1. Overview

Fumadocs ships **Orama** as the default search engine, configured out of the box:

- **Server endpoint** at `/api/search` (returns search results via API).
- **Static JSON file** mode for fully static deployments (no runtime).
- **Client UI** using Fumadocs' `<SearchDialog>` component.
- **i18n support** via `useI18n` + per-locale indexing.
- **Tag filtering** UI for filtering results by taxonomy.

The current `documentation-template` ships the **fetch mode** with a basic search dialog (already configured in `src/app/api/search/route.ts`).

---

## 2. Current state in `documentation-template`

`src/app/api/search/route.ts`:

```ts
import { source } from '@/lib/source';
import { createFromSource } from 'fumadocs-core/search/server';

export const { GET } = createFromSource(source, {
  language: 'english',
});
```

This is the **fetch mode** — every search query hits a Next.js API route. Fine for most deployments, but doesn't work for purely static export.

---

## 3. Custom search dialog

The default Fumadocs UI dialog works, but for full control, recreate it:

`src/components/search.tsx`:

```tsx
'use client';
import { useDocsSearch } from 'fumadocs-core/search/client';
import {
  SearchDialog,
  SearchDialogClose,
  SearchDialogContent,
  SearchDialogHeader,
  SearchDialogIcon,
  SearchDialogInput,
  SearchDialogList,
  SearchDialogOverlay,
  type SharedProps,
} from 'fumadocs-ui/components/dialog/search';
import { useI18n } from 'fumadocs-ui/contexts/i18n';
import { fetchClient } from 'fumadocs-core/search/client/fetch';

export default function DefaultSearchDialog(props: SharedProps) {
  const { locale } = useI18n(); // optional, for i18n
  const { search, setSearch, query } = useDocsSearch({
    client: fetchClient({ locale }),
  });

  return (
    <SearchDialog
      search={search}
      onSearchChange={setSearch}
      isLoading={query.isLoading}
      {...props}
    >
      <SearchDialogOverlay />
      <SearchDialogContent>
        <SearchDialogHeader>
          <SearchDialogIcon />
          <SearchDialogInput />
          <SearchDialogClose />
        </SearchDialogHeader>
        <SearchDialogList items={query.data !== 'empty' ? query.data : null} />
      </SearchDialogContent>
    </SearchDialog>
  );
}
```

Then wire it via the `RootProvider`:

```tsx
import { RootProvider } from 'fumadocs-ui/provider/next';
import SearchDialog from '@/components/search';

<RootProvider search={{ SearchDialog }}>
  {children}
</RootProvider>;
```

If `RootProvider` is in a server component (which it is in `src/app/layout.tsx`), you need a thin client wrapper:

`src/components/provider.tsx`:
```tsx
'use client';
import { RootProvider } from 'fumadocs-ui/provider/next';
import SearchDialog from '@/components/search';
import type { ReactNode } from 'react';

export function Provider({ children }: { children: ReactNode }) {
  return (
    <RootProvider search={{ SearchDialog }}>
      {children}
    </RootProvider>
  );
}
```

Then `src/app/layout.tsx` (or `[lang]/layout.tsx`) imports `Provider` instead of `RootProvider` directly.

---

## 4. Tag filtering — search by category

Useful for filtering by tags like "Guide", "API", "Tutorial":

```tsx
'use client';

import {
  SearchDialog,
  SearchDialogContent,
  SearchDialogFooter,
  SearchDialogOverlay,
  type SharedProps,
  TagsList,
  TagsListItem,
} from 'fumadocs-ui/components/dialog/search';
import { useState } from 'react';
import { useDocsSearch } from 'fumadocs-core/search/client';
import { fetchClient } from 'fumadocs-core/search/client/fetch';

export default function CustomSearchDialog(props: SharedProps) {
  const [tag, setTag] = useState<string | undefined>();
  const { search, setSearch, query } = useDocsSearch({
    client: fetchClient({ tag }),
  });

  return (
    <SearchDialog
      search={search}
      onSearchChange={setSearch}
      isLoading={query.isLoading}
      {...props}
    >
      <SearchDialogOverlay />
      <SearchDialogContent>
        {/* ... header, input, list ... */}
        <SearchDialogFooter className="flex flex-row">
          <TagsList tag={tag} onTagChange={setTag}>
            <TagsListItem value="guide">Guide</TagsListItem>
            <TagsListItem value="api">API</TagsListItem>
            <TagsListItem value="tutorial">Tutorial</TagsListItem>
          </TagsList>
        </SearchDialogFooter>
      </SearchDialogContent>
    </SearchDialog>
  );
}
```

For this to work, tag values must be indexed server-side. Configure the source to expose tags (via the frontmatter schema or a custom field).

---

## 5. i18n integration

Once i18n is enabled (issue #10):

```tsx
const { locale } = useI18n();
const { search, setSearch, query } = useDocsSearch({
  client: fetchClient({ locale }),  // server filters by locale
});
```

Server-side, `createFromSource` indexes per locale when `i18n` is passed to the source loader (per issue #10).

The per-locale indexing means:
- A French user searching sees French results.
- Search results respect the active route's locale prefix.

---

## 6. Static mode — for full static export

If the adopter wants `output: 'export'` (purely static), the API route approach doesn't work. Switch to **static mode**:

```ts
// next.config.mjs
const config = {
  output: 'export',  // static
};
```

The Orama index becomes a JSON file generated at build time:

```ts
// src/app/api/search/static.json.ts (or similar)
import { source } from '@/lib/source';
import { createFromSource } from 'fumadocs-core/search/server';

export const revalidate = false;

export async function GET() {
  return new Response(JSON.stringify(await createStaticIndex(source)), {
    headers: { 'Content-Type': 'application/json' },
  });
}
```

Use `staticClient` on the client side instead of `fetchClient`.

**Tradeoff**: static mode requires rebuilding on every content change. For docs sites with infrequent updates, this is fine.

---

## 7. Language support

Orama supports multiple languages for tokenization and stemming:

```ts
export const { GET } = createFromSource(source, {
  language: 'english',  // or 'french', 'german', 'spanish', etc.
});
```

For multi-locale docs (i18n — issue #10):

```ts
export const { GET } = createFromSource(source, ({ locale }) => {
  return {
    language: locale === 'fr' ? 'french' : 'english',
  };
});
```

Orama's language option affects:
- Stemming ("running" matches "run")
- Stop word filtering ("the", "a", "an")
- Tokenization rules

---

## 8. Algolia — alternative

If the docs site has **high search volume** or needs **AI-powered Q&A**, swap Orama for Algolia DocSearch:

- Algolia provides official multilingual support.
- Requires a paid Algolia account (free tier available for OSS).
- Different API; Fumadocs has a separate integration.

For `documentation-template`, **stay on Orama by default**. Document Algolia as an optional upgrade in CONTRIBUTING.md.

---

## 9. Specific considerations for `documentation-template`

### Current state

- ✅ Server-side search via `createFromSource` in `src/app/api/search/route.ts`
- ✅ Default Fumadocs search dialog
- ❌ No client `Provider` wrapper (using `RootProvider` directly)
- ❌ No tag filtering
- ❌ No custom search UI

### Recommended upgrades (optional)

1. **Wrap `RootProvider` in a client `Provider`** to enable custom search dialog overrides later. ~30 min change.
2. **Add tag filtering** once pages have taxonomy (post i18n + blog). Easier after content structure stabilizes.
3. **Consider static mode** if the adopter targets pure static export (Cloudflare Pages, GitHub Pages). Affects the entire stack, not just search.

### Interactions with other issues

- **#10 i18n**: search needs per-locale indexing. Update `createFromSource` to use a function form.
- **#1 blog + #2 changelog**: search should include blog/changelog content. Multi-source search needs `createSearchAPI` or similar. Verify against current Fumadocs docs.
- **#11 API reference**: OpenAPI endpoints should be searchable. `staticSource` handles this automatically.

---

## 10. Open questions

- [ ] Does the default Fumadocs search dialog (without custom override) work with i18n out of the box?
- [ ] How does multi-source search (docs + blog + changelog + OpenAPI) compose?
- [ ] What's the recommended approach for search analytics (privacy-friendly)?
- [ ] Does `useDocsSearch` debounce input, or is it the caller's responsibility?
- [ ] Does Algolia DocSearch support the same `tag` API as Orama's tag filter?

---

## 11. References

- Official docs: <https://www.fumadocs.dev/docs/search/orama>
- Built-in Search API: <https://fumadocs.dev/docs/headless/search>
- Algolia integration: <https://fumadocs.dev/docs/headless/search/algolia>
- Orama Cloud: <https://fumadocs.dev/docs/headless/search/orama-cloud>
- Sibling issue: #10 (i18n) — per-locale indexing
- Sibling issue: #11 (API reference) — OpenAPI endpoint indexing
- Sibling issue: #1 (blog) + #2 (changelog) — multi-source search
