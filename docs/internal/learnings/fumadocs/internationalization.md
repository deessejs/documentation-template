# Fumadocs Internationalization (i18n) for Next.js

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/internationalization/next). Captured for the `documentation-template` project so we can refer back when implementing issue #10.

---

## 1. Overview

Fumadocs ships **first-class i18n support** for Next.js apps. It's not a wrapper around `next-intl` or `next-i18next` — it's a built-in convention that integrates directly with the Fumadocs source loader, layouts, and search.

The model is:

- A single `defineI18n()` config declares your supported languages.
- A middleware redirects bare URLs (`/docs`) to the locale-prefixed form (`/en/docs` or `/docs`, depending on `hidePrefix`).
- Routes are organized under `/app/[lang]/...` so every layout and page receives the locale via `params`.
- The source loader accepts the locale to fetch the right content tree.
- Fumadocs UI strings are translated via a `translations` object that the root layout passes to `RootProvider` via `i18nProvider()`.

This gives you **locale-prefixed URLs**, **per-locale page trees**, **per-locale search**, and **translatable UI strings** without bolting on a third-party i18n library.

---

## 2. When to use it

Use Fumadocs i18n when:

- You need more than one language for your docs.
- You're willing to use Next.js's `[lang]` segment convention (path-based i18n).
- You're not already invested in another i18n library.

Do **not** use it if:

- You want domain-based i18n (`fr.example.com`) — Fumadocs assumes path-based.
- You need locale-based content negotiation (e.g. `Accept-Language` headers without URL changes) — the middleware model is URL-driven.
- You're happy with a single language forever — single-locale docs don't need any of this complexity.

---

## 3. Setup — i18n config

Create a single config file. The convention is `src/lib/i18n.ts`.

```ts
import { defineI18n } from 'fumadocs-core/i18n';

export const i18n = defineI18n({
  defaultLanguage: 'en',
  languages: ['en', 'cn'],
});
```

Available options (from the `defineI18n` API):

| Option | Type | Purpose |
|---|---|---|
| `defaultLanguage` | `string` | Locale used when no prefix is in the URL (also the fallback). |
| `languages` | `string[]` | All supported locales. Order is preserved in the locale switcher. |
| `hidePrefix` | `string \| 'always' \| 'never'` | Hide the prefix for the default language (`'always'`) or for specific locales. |
| `locale` | custom | A function returning a locale object with `name`, `label`, etc. |

For `documentation-template`, the natural initial config is:

```ts
export const i18n = defineI18n({
  defaultLanguage: 'en',
  languages: ['en', 'fr'], // English + French as a demonstration
  hidePrefix: 'always',    // /docs serves English without prefix
});
```

`hidePrefix: 'always'` means:
- `/docs` → English (no prefix shown)
- `/fr/docs` → French (prefix shown)

This is what most public docs sites use — the default language looks "clean" while non-default languages are clearly tagged.

---

## 4. Middleware

Create `src/middleware.ts` to redirect bare URLs to the correct locale:

```ts
import { createI18nMiddleware } from 'fumadocs-core/i18n/middleware';
import { i18n } from '@/lib/i18n';

export default createI18nMiddleware(i18n);

export const config = {
  // Matcher ignoring `/_next/`, `/api/`, static assets
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
};
```

### Caveat: existing `proxy.ts`

This template already ships a **`proxy.ts`** for Markdown content negotiation (the `proxy` export serves `/docs/foo.md` as Markdown when `Accept: text/markdown`). Two middlewares/proxies can't coexist cleanly. You have two options:

1. **Extend the existing `proxy.ts`** with i18n redirect logic (preferred — keeps one middleware).
2. **Use Fumadocs' middleware and merge the content-negotiation rewrite** into it.

Option 1 sketch:

```ts
import { NextRequest, NextResponse } from 'next/server';
import { isMarkdownPreferred, rewritePath } from 'fumadocs-core/negotiation';
import { createI18nMiddleware } from 'fumadocs-core/i18n/middleware';
import { i18n } from '@/lib/i18n';

const i18nMiddleware = createI18nMiddleware(i18n);

const { rewrite: rewriteDocs } = rewritePath(/* ... */);
const { rewrite: rewriteSuffix } = rewritePath(/* ... */);

export default function proxy(request: NextRequest) {
  // Content negotiation first (most specific)
  const suffixResult = rewriteSuffix(request.nextUrl.pathname);
  if (suffixResult) {
    return NextResponse.rewrite(new URL(suffixResult, request.nextUrl));
  }

  if (isMarkdownPreferred(request)) {
    const result = rewriteDocs(request.nextUrl.pathname);
    if (result) {
      return NextResponse.rewrite(new URL(result, request.nextUrl));
    }
  }

  // Fallback to i18n redirect
  return i18nMiddleware(request);
}
```

---

## 5. Translations

Two kinds of strings to translate:

1. **Fumadocs UI strings** (search placeholder, navigation labels, theme toggle, etc.) — provided by `fumadocs-ui/i18n`.
2. **App-specific strings** (display names, custom labels) — defined per-locale.

Setup in `src/lib/layout.shared.tsx`:

```tsx
import { i18n } from '@/lib/i18n';
import { uiTranslations } from 'fumadocs-ui/i18n';
import type { BaseLayoutProps } from 'fumadocs-ui/layouts/shared';

export const translations = i18n
  .translations()
  .extend(uiTranslations())
  .add({
    en: {
      displayName: 'English',
    },
    fr: {
      displayName: 'French',
    },
  });

// baseOptions now takes a locale parameter
export function baseOptions(locale: string): BaseLayoutProps {
  return {
    nav: {
      title: 'Documentation Template', // could vary by locale
    },
    githubUrl: `https://github.com/deessejs/documentation-template`,
    i18n: locale, // propagates locale to layout components
  };
}
```

Note: this changes the signature of `baseOptions()` from `baseOptions()` to `baseOptions(locale: string)`. Every layout call site must be updated.

---

## 6. Routing — `/app/[lang]/...`

Move all route files under a `[lang]` segment:

```
src/app/
├── [lang]/
│   ├── (home)/
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── docs/
│   │   ├── layout.tsx
│   │   └── [[...slug]]/
│   │       └── page.tsx
│   ├── api/
│   │   └── search/
│   │       └── route.ts
│   ├── og/
│   │   └── docs/[...slug]/
│   │       └── route.tsx
│   └── llms-full.txt/
│       └── route.ts
├── llms.txt/
│   └── route.ts
├── llms.mdx/
│   └── docs/[[...slug]]/
│       └── route.ts
└── middleware.ts
```

**Important**: route handlers (`route.ts`/`route.tsx`) DO NOT receive `params.lang` — they handle their own locale resolution via `request` (e.g., read it from the URL).

**Common mistake** (per Fumadocs docs): if you move pages without moving `global.css` properly, your styles break. Verify the import path in `[lang]/layout.tsx` still resolves to `src/app/global.css`.

---

## 7. Source loader integration

Update `src/lib/source.ts` to accept the i18n config:

```ts
import { docs } from 'collections/server';
import { loader } from 'fumadocs-core/source';
import { lucideIconsPlugin } from 'fumadocs-core/source/lucide-icons';
import { i18n } from '@/lib/i18n';

export const source = loader({
  i18n,                          // ← NEW: pass i18n to loader
  baseUrl: '/docs',
  source: docs.toFumadocsSource(),
  plugins: [lucideIconsPlugin()],
});
```

The loader will now:
- Generate routes under `[lang]`.
- Filter `getPages()` / `getPage()` by locale when called with a `lang` argument.

---

## 8. Layouts and pages

### Root layout — pass locale to `RootProvider`

```tsx
import { RootProvider } from 'fumadocs-ui/provider/next';
import { Inter } from 'next/font/google';
import { translations } from '@/lib/layout.shared';
import { i18nProvider } from 'fumadocs-ui/i18n';

const inter = Inter({ subsets: ['latin'] });

export default async function RootLayout({
  params,
  children,
}: {
  params: Promise<{ lang: string }>;
  children: React.ReactNode;
}) {
  const { lang } = await params;

  return (
    <html lang={lang} className={inter.className} suppressHydrationWarning>
      <body className="flex flex-col min-h-screen">
        <RootProvider i18n={i18nProvider(translations, lang)}>
          {children}
        </RootProvider>
      </body>
    </html>
  );
}
```

### Home layout

```tsx
import type { ReactNode } from 'react';
import { HomeLayout } from 'fumadocs-ui/layouts/home';
import { baseOptions } from '@/lib/layout.shared';

export default async function Layout({
  params,
  children,
}: {
  params: Promise<{ lang: string }>;
  children: ReactNode;
}) {
  const { lang } = await params;
  return <HomeLayout {...baseOptions(lang)}>{children}</HomeLayout>;
}
```

### Docs layout — locale-aware page tree

```tsx
import type { ReactNode } from 'react';
import { source } from '@/lib/source';
import { DocsLayout } from 'fumadocs-ui/layouts/docs';
import { baseOptions } from '@/lib/layout.shared';

export default async function Layout({
  params,
  children,
}: {
  params: Promise<{ lang: string }>;
  children: ReactNode;
}) {
  const { lang } = await params;
  return (
    <DocsLayout {...baseOptions(lang)} tree={source.getPageTree(lang)}>
      {children}
    </DocsLayout>
  );
}
```

### Docs page — pass locale to source methods

```ts
import { source } from '@/lib/source';

export default async function Page({
  params,
}: {
  params: Promise<{ lang: string; slug?: string[] }>;
}) {
  const { slug, lang } = await params;

  // Before i18n: source.getPage(slug)
  // After i18n:
  const page = source.getPage(slug, lang);
  // ...
}
```

Same for `getPages()`:

```ts
// Before: source.getPages()
// After:
source.getPages(lang);
```

### `generateStaticParams` — non-`lang` segment name

If you used `[locale]` instead of `[lang]`, update:

```ts
export function generateStaticParams() {
  // Default uses 'slug' and 'lang' as the param names
  return source.generateParams('slug', 'locale');
}
```

---

## 9. Search

### Orama (built-in)

Already configured via `createFromSource(source, ...)`. Pass `i18n` to the source loader (see §7), and Orama indexes per-locale automatically.

See the [Orama i18n docs](https://www.fumadocs.dev/docs/headless/search) for the per-locale language option (`language: 'english'` becomes `language: 'english'` for English pages, `language: 'french'` for French pages).

### Algolia DocSearch

Officially supports multilingual — configure one index per locale per the Algolia docs.

---

## 10. Writing documents — content tree

Move existing content under locale roots:

```
content/
└── docs/
    ├── (en)/
    │   ├── index.mdx
    │   ├── components.mdx
    │   └── meta.json
    └── (fr)/
        ├── index.mdx
        ├── components.mdx
        └── meta.json
```

Each locale has its own `meta.json` for sidebar ordering. The folder names `(en)` and `(fr)` are **convention** — Fumadocs treats them as locale roots when `i18n` is configured.

`source.config.ts` may need to be updated to declare multiple collections:

```ts
export const docs = defineDocs({
  dir: 'content/docs', // still the root
  // docs collection auto-detects (en)/(fr) subdirs
});
```

(Verify against current Fumadocs docs — this may have evolved.)

---

## 11. Navigation — locale-aware links

Fumadocs handles the **sidebar** automatically. For custom links, you must prepend the locale manually.

### In React components

```tsx
import Link from 'next/link';
import { useParams } from 'next/navigation';

const { lang } = useParams();
return <Link href={`/${lang}/another-page`}>Another page</Link>;
```

### In MDX content

Use `DynamicLink` from `fumadocs-core/dynamic-link`:

```mdx
import { DynamicLink } from 'fumadocs-core/dynamic-link';

<DynamicLink href="/[lang]/another-page">Another page</DynamicLink>
```

The `[lang]` token is replaced at build time with the active locale.

---

## 12. SEO implications

i18n directly couples with the **rich metadata issue (#4)**:

- **`hreflang` tags**: `alternates.languages` in `generateMetadata` should list every supported locale with its URL.
- **Canonical URL**: per-locale, not global.
- **OG/Twitter metadata**: optionally localized title/description per locale.
- **Sitemap** (#3): should group pages by locale and reference the locale switcher.

Skeleton:

```ts
alternates: {
  canonical: page.url,
  languages: {
    en: `${baseUrl}/en${page.url}`,
    fr: `${baseUrl}/fr${page.url}`,
  },
}
```

---

## 13. Specific considerations for `documentation-template`

When implementing i18n on this template, the **migration order** matters:

1. **Create `src/lib/i18n.ts`** (no breaking change).
2. **Update `src/lib/source.ts`** to pass `i18n` to `loader`.
3. **Restructure `src/app/`** under `[lang]/...` (biggest change).
4. **Update `src/lib/layout.shared.tsx`** signature — `baseOptions(locale)`.
5. **Update `src/app/layout.tsx`** to be inside `[lang]/layout.tsx`.
6. **Merge i18n middleware into `proxy.ts`** (don't add a second middleware).
7. **Move `content/docs/` into `content/docs/(en)/`** and add `(fr)/` mirror.
8. **Update `package.json` scripts** if any reference paths.
9. **Update `CONTRIBUTING.md`** with locale workflow.
10. **Update `README.md`** Customizing section.

### Estimated impact

- **Files touched**: ~15-20
- **New files**: ~5 (i18n config, middleware, French translations, sample French content)
- **Effort**: 1-2 days for an experienced contributor
- **Risk**: Medium — the existing `proxy.ts` is the trickiest part

---

## 14. Open questions / things to verify

- [ ] Does the current `source.config.ts` auto-detect `(en)/(fr)` subdirs, or does it need explicit collections?
- [ ] Should `defaultLanguage` be `'en'` or `'en-US'`? (Locale code format affects URLs.)
- [ ] Does Orama need per-locale `language` config, or is it auto-detected?
- [ ] How does `hidePrefix: 'always'` interact with `generateStaticParams`?
- [ ] What's the migration path for adopters who already have `/docs/foo` URLs indexed by Google? (301 redirects needed.)

---

## 15. References

- Official docs: <https://www.fumadocs.dev/docs/internationalization/next>
- Next.js i18n guide: <https://nextjs.org/docs/app/building-your-application/routing/internationalization>
- Issue #10 in this repo: feat(docs): add internationalization (i18n) support
- Sibling issues to coordinate with: #9 (versioning), #4 (rich metadata for hreflang)
