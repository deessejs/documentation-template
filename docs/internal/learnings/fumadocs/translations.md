# Fumadocs UI Translations

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/ui/translations). Complements `internationalization.md` — this doc covers the **UI strings** layer (search placeholder, nav labels, theme toggle, etc.), which sits on top of the i18n routing setup.

---

## 1. Overview

All official Fumadocs packages ship in **English by default**. The translation system is layered:

```
┌────────────────────────────────────────────────┐
│  Content (MDX)                                  │  ← handled by i18n routing
├────────────────────────────────────────────────┤
│  UI strings (search box, nav, theme toggle)     │  ← THIS DOC
├────────────────────────────────────────────────┤
│  Fumadocs core / UI / OpenAPI integrations      │  ← registered via "extensions"
└────────────────────────────────────────────────┘
```

The translation API is centered on the **`Translation`** type from `fumadocs-core/i18n`, with three operations:

| Operation | Purpose |
|---|---|
| `extend(other)` | Merge in keys from another extension (Fumadocs UI, OpenAPI, custom) |
| `add({ ... })` | Provide values for the registered keys |
| `preset(pack)` | Consume a pre-built language pack in one call |

---

## 2. Two modes — singular vs multilingual

### Singular — translate into ONE language

Useful when you want a single non-English locale (no switching):

```ts
// src/lib/layout.shared.tsx
import { defineTranslations } from 'fumadocs-core/i18n';
import { uiTranslations } from 'fumadocs-ui/i18n';

export const translations = defineTranslations()
  .extend(uiTranslations())
  .add({
    'Search(search trigger)': '搜尋文檔',
  });
```

Wire in the root layout:

```tsx
import { RootProvider } from 'fumadocs-ui/provider/next';
import { i18nProvider } from 'fumadocs-ui/i18n';
import { translations } from '@/lib/layout.shared';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <RootProvider i18n={i18nProvider(translations)}>
      {children}
    </RootProvider>
  );
}
```

Note: `i18nProvider(translations)` — **no locale argument** for singular.

### Multilingual — multiple languages via i18n routing

Requires the full i18n setup from `internationalization.md`:

```ts
// src/lib/layout.shared.tsx
import { i18n } from '@/lib/i18n';
import { uiTranslations } from 'fumadocs-ui/i18n';

export const translations = i18n
  .translations()
  .extend(uiTranslations())
  .add({
    cn: {
      'Search(search trigger)': '搜尋文檔',
    },
  });
```

Wire in the root layout with the locale param:

```tsx
export default function RootLayout({
  locale,
  children,
}: {
  locale: string;
  children: React.ReactNode;
}) {
  return (
    <RootProvider i18n={i18nProvider(translations, locale)}>
      {children}
    </RootProvider>
  );
}
```

Note: `i18nProvider(translations, locale)` — **locale required**.

---

## 3. The `.add()` ordering rule

> `.add()` should only be called **after** all `.extend()` calls.

```ts
// ✅ Correct
defineTranslations()
  .extend(uiTranslations())
  .extend(openapiTranslations())
  .add({ ... });

// ❌ Wrong — adds before extends register their keys
defineTranslations()
  .add({ ... })
  .extend(uiTranslations());
```

Why: `.extend()` registers the **available keys**. `.add()` validates against those keys. Calling `.add()` early means your translations may be rejected as "unknown labels".

---

## 4. Label format — `"label(context)"`

Fumadocs uses a structured key format:

```
"label text(additional context)"
```

Examples from the Fumadocs codebase:

```
'Search(search trigger)'
'loading...(playground server select)'
'No Variant(story controls)'
'Hello {user}(welcome screen)'
```

The `(context)` part disambiguates when the same English phrase appears in multiple UI surfaces. Without it, the translation pipeline can't tell them apart.

### Placeholders

Some keys contain template variables like `{user}`. Translations must preserve them:

```ts
.add({
  'Hello {user}(welcome screen)': '你好 {user}',
});
```

Skipping the placeholder silently breaks the rendered text.

---

## 5. Language packs — `@fumadocs/language`

Pre-built translations for popular languages, maintained by the Fumadocs team.

### Install

```bash
pnpm add @fumadocs/language
```

### Use via `preset()`

```ts
import { defineTranslations } from 'fumadocs-core/i18n';
import { uiTranslations } from 'fumadocs-ui/i18n';
import { openapiTranslations } from 'fumadocs-openapi/i18n';
import { zhTW } from '@fumadocs/language/zh-tw';

export const translations = defineTranslations()
  .extend(uiTranslations())
  // add extensions according to the integrations you configured
  .extend(openapiTranslations())
  // Traditional Chinese
  .preset(zhTW());
```

### Available packs (verify current list at `@fumadocs/language` README)

The package ships per-locale subpath imports:

- `@fumadocs/language/zh-cn` (Simplified Chinese)
- `@fumadocs/language/zh-tw` (Traditional Chinese)
- `@fumadocs/language/ja` (Japanese)
- `@fumadocs/language/ko` (Korean)
- `@fumadocs/language/fr` (French)
- `@fumadocs/language/de` (German)
- `@fumadocs/language/es` (Spanish)
- `@fumadocs/language/pt-br` (Brazilian Portuguese)
- `@fumadocs/language/ru` (Russian)
- `@fumadocs/language/ar` (Arabic)
- `@fumadocs/language/it` (Italian)
- `@fumadocs/language/nl` (Dutch)
- `@fumadocs/language/pl` (Polish)
- `@fumadocs/language/tr` (Turkish)
- `@fumadocs/language/vi` (Vietnamese)
- `@fumadocs/language/id` (Indonesian)

(French, German, etc. — verify against `@fumadocs/language` repo for current coverage.)

### If a language is missing

The Fumadocs team welcomes PRs to add new language packs. See "Custom Packs" below for how to author one.

---

## 6. Custom packs — build your own

For a language not covered by the official packs, or for an organization-specific vocabulary:

### Pack structure

```ts
// packages/my-lang-pack/src/index.ts
import type { TranslationPreset } from 'fumadocs-core/i18n';
import * as OpenAPI from 'fumadocs-openapi/i18n';
import * as UI from 'fumadocs-ui/i18n';
import * as Story from '@fuma-docs/story/i18n';

const translations = {
  displayName: '繁體中文',
  'loading...(playground server select)': '載入中...',
  'No Variant(story controls)': '沒有變體',
  // ... exhaustive list of every key from each extension
} satisfies Partial<UI.Translations & OpenAPI.Translations & Story.Translations>;

export function zhTW(): TranslationPreset<keyof typeof translations> {
  return {
    name: 'zh-TW',
    value: translations,
  };
}
```

### Publish as npm package

A custom pack can be:
- A **standalone npm package** (recommended for organizations with many adopters).
- An **inline module** within the repo (for single-team use).

For `documentation-template`, an inline module at `src/lib/i18n/packs/fr.ts` is sufficient for the demonstration French translation.

### `satisfies Partial<...>`

The `Partial<UI.Translations & OpenAPI.Translations & Story.Translations>` constraint:
- Forces you to handle only the keys you care about.
- TypeScript catches typos in key names.
- TypeScript catches missing translations for keys you DO cover.

---

## 7. Extensions — declaring available keys

Fumadocs components call `useTranslations('namespace')` internally to render their strings. Each namespace corresponds to an **extension** that declares the keys.

### Built-in extensions

| Extension | Source | Keys (subset) |
|---|---|---|
| `uiTranslations()` | `fumadocs-ui/i18n` | Search, theme toggle, navigation, TOC, etc. |
| `openapiTranslations()` | `fumadocs-openapi/i18n` | API playground labels |
| Story book | `@fumadocs/story/i18n` | Story-specific labels |

### Why extensions matter

> Only registered labels are sent to the client payload.

If a Fumadocs component calls `t('Search(search trigger)')` but you didn't `.extend(uiTranslations())`, that label won't be in the client bundle — and either falls back to English (silent) or fails (loud, depending on the implementation).

For `documentation-template`, this means **you must extend `uiTranslations()` even if you don't add any custom translations**, otherwise the default English strings get out of sync with newer Fumadocs versions.

### Custom extension — for community packages

```ts
// your-community-package/src/i18n.ts
import type { TranslationExtension } from 'fumadocs-core/i18n';

const keys = [
  // key format: "your label(additional context)"
  'Hello {user}(welcome screen)',
] as const;

export function myTranslations(): TranslationExtension<(typeof keys)[number]> {
  return { keys };
}
```

Consumers then:

```ts
import { defineTranslations } from 'fumadocs-core/i18n';
import { myTranslations } from 'my-package/i18n';

export const translations = defineTranslations()
  .extend(myTranslations())
  .add({
    'Hello {user}(welcome screen)': '你好 {user}',
  });
```

### Advanced — Fuma Translate

For larger projects, the `@fuma-translate/react` toolkit provides:
- Compile-time translation key generation (no hardcoded strings).
- Type-safe translation calls.
- Plural and gender forms.

The Fumadocs docs link to it but defer details to its own README.

---

## 8. Integration with i18n routing

The two docs (`internationalization.md` and this one) cover complementary concerns:

| Concern | Doc |
|---|---|
| URL structure (`/fr/docs/...`) | `internationalization.md` |
| Locale detection middleware | `internationalization.md` |
| Source loader per-locale pages | `internationalization.md` |
| Per-locale MDX content | `internationalization.md` |
| **Fumadocs UI string translations** | **THIS DOC** |
| **Language packs and extensions** | **THIS DOC** |
| Custom UI translations for adopters | **THIS DOC** |

For `documentation-template` to ship i18n, **both docs need to be applied together**.

---

## 9. Specific considerations for `documentation-template`

### Current state

- No i18n config, no translations.
- All Fumadocs UI strings render in English (default).
- `RootProvider` is used in `src/app/layout.tsx` without `i18n` prop → English defaults work.

### What changes when adopting i18n + translations

1. Create `src/lib/i18n.ts` (per `internationalization.md` §3).
2. Modify `src/lib/layout.shared.tsx` — add `translations` export and `baseOptions(locale: string)`.
3. Modify `src/app/layout.tsx` → `src/app/[lang]/layout.tsx` — pass `i18n` to `RootProvider`.
4. Add a translation file (e.g. `src/lib/i18n/translations/fr.ts`) or use a language pack.

### Minimal viable French setup

```ts
// src/lib/layout.shared.tsx
import { i18n } from '@/lib/i18n';
import { uiTranslations } from 'fumadocs-ui/i18n';

export const translations = i18n
  .translations()
  .extend(uiTranslations())
  .add({
    fr: {
      displayName: 'Français',
      'Search(search trigger)': 'Rechercher',
      // ...adopters extend this as needed
    },
  });

export function baseOptions(locale: string) {
  return {
    nav: { title: 'Documentation Template' },
    githubUrl: 'https://github.com/deessejs/documentation-template',
    i18n: locale,
  };
}
```

### Recommendations for the template

1. **Always `.extend(uiTranslations())`** even when no custom translations — keeps parity with upstream.
2. **Use language packs** (e.g. `@fumadocs/language/fr`) when available — avoids maintaining translations yourself.
3. **Demonstrate overrides** for at least 2-3 keys (search, navigation, theme toggle) so adopters see the pattern.
4. **Document the extension map** in `CONTRIBUTING.md` — adopters must extend when adding custom Fumadocs components.

### Estimated effort

- **New files**: `src/lib/i18n.ts`, optionally `src/lib/i18n/translations/fr.ts`
- **Modified files**: `src/lib/layout.shared.tsx`, `src/app/layout.tsx` (or `[lang]/layout.tsx`)
- **Effort**: ~1 hour for a singular French setup, ~3-4 hours for multilingual with i18n routing
- **Dependencies**: `@fumadocs/language` (optional, ~50KB added to node_modules)

---

## 10. Open questions

- [ ] Does `defineTranslations()` differ from `i18n.translations()` in current Fumadocs versions, or are they aliases?
- [ ] Is `fumadocs-ui` v16+ still using the `useTranslations` pattern, or has the API changed?
- [ ] Does `@fumadocs/language` ship **all** UI strings or only a subset? (Verification needed.)
- [ ] For custom packs, can keys from multiple extensions be merged in one `.add()` call (current docs show single-extension usage)?
- [ ] Does `i18nProvider` fall back gracefully to English when a translation is missing, or render `undefined`?

---

## 11. References

- Official docs: <https://www.fumadocs.dev/docs/ui/translations>
- i18n routing: <https://www.fumadocs.dev/docs/internationalization/next>
- `@fumadocs/language`: <https://github.com/fuma-nama/fumadocs/tree/main/packages/language>
- `fumadocs-core/i18n` API: <https://fumadocs.dev/docs/headless/internationalization>
- `@fuma-translate/react`: <https://github.com/fuma-nama/fuma-translate>
- Sibling learnings: `docs/internal/learnings/fumadocs/internationalization.md`
