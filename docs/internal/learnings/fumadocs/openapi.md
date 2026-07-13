# Fumadocs OpenAPI Integration

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/integrations/openapi). Reference for the `fumadocs-openapi` package — directly relevant for issue #11 (auto-generated API reference).

---

## 1. Overview

Fumadocs ships a first-class OpenAPI integration via the **`fumadocs-openapi`** package:

- Reads an OpenAPI 3 schema (JSON or YAML).
- Renders a **Swagger-like UI** with interactive playground inside Fumadocs UI.
- Generates example code in multiple programming languages.
- Supports per-endpoint pages with parameters, body, response samples, and TypeScript definitions.

Two integration modes:
1. **Generated MDX files** — `generateFiles()` writes MDX once; pages look like hand-written content.
2. **Static source** — `staticSource()` injects OpenAPI endpoints into the `loader()` so they coexist with MDX docs and the page tree reflects them.

For `documentation-template`, the **static source** mode is more idiomatic (the page tree automatically includes API endpoints as if they were MDX files).

---

## 2. Setup

### Install

```bash
pnpm add fumadocs-openapi shiki
```

### CSS

Add to `src/app/global.css`:

```css
@import 'tailwindcss';
@import 'fumadocs-ui/css/neutral.css';
@import 'fumadocs-ui/css/preset.css';
@import 'fumadocs-openapi/css/preset.css';
```

### Configure the OpenAPI server (`src/lib/openapi.ts`)

```ts
import { createOpenAPI } from 'fumadocs-openapi/server';

// Server-side API
export const openapi = createOpenAPI({
  input: ['./openapi.json'],  // or 'https://api.example.com/openapi.json'
});
```

### Client UI component (`src/components/api-page.tsx`)

```tsx
'use client';
import { createOpenAPIPage } from 'fumadocs-openapi/ui';

export const OpenAPIPage = createOpenAPIPage();
```

This single component handles the full Swagger-like UI: endpoint info, parameters, request body, responses, code samples, and the playground.

### Add plugin to the source loader (`src/lib/source.ts`)

```ts
import { openapiPlugin } from 'fumadocs-openapi/server';

export const source = loader({
  // ...existing options...
  plugins: [openapiPlugin()],  // adds a badge to each OpenAPI page in the tree
});
```

---

## 3. Two generation modes

### Mode A — Generated MDX files

For projects where the API spec rarely changes and you want pages to look exactly like hand-written MDX:

`scripts/generate-docs.ts`:
```ts
import { generateFiles } from 'fumadocs-openapi';
import { openapi } from '@/lib/openapi';

void generateFiles({
  input: openapi,
  output: './content/docs',
  includeDescription: true,
});
```

Run with Bun or tsx:
```bash
pnpm tsx scripts/generate-docs.ts
```

Commit the generated MDX to the repo (so PR diffs show API changes).

### Mode B — Static source (recommended for the template)

Pages are generated dynamically into the `loader()`. The page tree **automatically** includes OpenAPI endpoints:

```ts
import { loader } from 'fumadocs-core/source';
import { docs } from 'collections/server';
import { openapi } from '@/lib/openapi';

export const source = loader(
  {
    docs: docs.toFumadocsSource(),
    openapi: await openapi.staticSource({
      baseDir: 'openapi',  // folder under content/ where OpenAPI pages live
    }),
  },
  {
    baseUrl: '/docs',
    plugins: [openapi.loaderPlugin()],
  },
);
```

**Tradeoff**: pages are computed at build time, but the source `pages` array now has a `type` discriminator: `page.type === 'docs'` or `page.type === 'openapi'`. All references to `page` must handle both.

---

## 4. Page rendering — branching on `page.type`

Once the source has mixed types, update consumers:

### Page renderer (`src/app/docs/[[...slug]]/page.tsx`)

```tsx
import { OpenAPIPage } from '@/components/api-page';

export default function Page({ params }) {
  const page = source.getPage(params.slug);

  // For OpenAPI pages
  if (page.type === 'openapi') {
    return (
      <DocsPage full>
        <DocsTitle>{page.data.title}</DocsTitle>
        <DocsDescription>{page.data.description}</DocsDescription>
        <DocsBody>
          <OpenAPIPage {...page.data.getOpenAPIPageProps()} />
        </DocsBody>
      </DocsPage>
    );
  }

  // Your original flow below...
  return <DocsBody>{/* MDX content */}</DocsBody>;
}
```

### LLM text endpoint (`src/lib/source.ts` → `getLLMText`)

```ts
export async function getLLMText(page) {
  if (page.type === 'openapi') {
    return JSON.stringify(page.data.getSchema().bundled, null, 2);
  }
  // ... existing flow for MDX pages
}
```

This makes the OpenAPI schema available to LLM crawlers — a powerful SEO/AI-search advantage (no competitor offers this out of the box).

---

## 5. Features supported

Per the official docs, the integration includes:

| Feature | Notes |
|---|---|
| Basic endpoint info | Method, path, description |
| Interactive playground | Try-it-out form for each endpoint |
| Multi-language examples | curl, JavaScript, Python, etc. |
| Response samples | Per status code |
| TypeScript definitions | Auto-generated from schema |
| Parameters + body | Rendered from OpenAPI schemas |
| Authentication display | Bearer, API key, OAuth flows |

A demo is available at https://fumadocs.vercel.app/openapi (verify URL in current docs).

---

## 6. Schema location — local vs remote

```ts
export const openapi = createOpenAPI({
  // Local file
  input: ['./openapi.json'],

  // Multiple files (merged)
  input: ['./api-v1.json', './api-v2.json'],

  // Remote URL
  input: ['https://api.example.com/openapi.json'],

  // Remote URL with auth
  input: [{ url: 'https://api.example.com/openapi.json', headers: { Authorization: `Bearer ${process.env.API_TOKEN}` } }],
});
```

For `documentation-template` adopters:
- **Local file** is the simplest. `openapi.yaml` at repo root.
- **Remote URL** is best for live API specs that may change between deploys. Requires CI/CD to refresh.

---

## 7. Specific considerations for `documentation-template`

### Recommended setup

1. **Mode B (static source)** — recommended. Page tree automatically includes endpoints.
2. **`openapi.yaml` at repo root** — easy to find, easy to validate.
3. **`prebuild` hook** in `package.json`:
   ```json
   "scripts": {
     "prebuild": "pnpm openapi:validate",
     "openapi:validate": "tsx scripts/validate-openapi.ts"
   }
   ```
4. **Validation** via `redocly/cli` or `@redocly/openapi-cli` to catch broken specs early.
5. **Example endpoint** that ships with the template — at least 2 endpoints (GET, POST) to demonstrate the UI.

### Estimated effort

- **New files**:
  - `src/lib/openapi.ts`
  - `src/components/api-page.tsx`
  - `openapi.yaml` (example)
  - `scripts/validate-openapi.ts` (optional)
- **Modified files**:
  - `src/lib/source.ts` (add `openapiPlugin` + `staticSource`)
  - `src/app/docs/[[...slug]]/page.tsx` (branch on `page.type`)
  - `src/app/global.css` (add `fumadocs-openapi/css/preset.css`)
  - `src/lib/source.ts` → `getLLMText` (handle `openapi` type)
  - `package.json` (add deps + scripts)
- **Effort**: ~4-6 hours for a contributor familiar with the codebase.

### Interactions with other issues

- **#11 API reference**: this is the implementation target. Static source mode (Mode B) is the recommended approach.
- **#8 shadcn**: shadcn + OpenAPI work well together (you can put example request bodies in a `<Card>` for visual emphasis).
- **#10 i18n**: `staticSource` should produce per-locale pages if i18n is active (verify against current docs).
- **#9 versioning**: each version can have its own `openapi.yaml` and source entry.
- **#4 rich metadata**: OpenAPI pages benefit from `alternates.canonical` (provided by the loader).
- **#3 sitemap**: OpenAPI endpoints should appear in `/sitemap.xml` (Fumadocs handles this if the source loader is registered).

---

## 8. Open questions

- [ ] Does `staticSource` re-generate the page tree on every build (slow) or cache between builds?
- [ ] How does `staticSource` interact with i18n routing — does each locale get its own OpenAPI tree?
- [ ] Can the OpenAPI UI be themed to match Fumadocs' default theme tokens, or is it locked to its own styles?
- [ ] What's the smallest viable `openapi.yaml` for the example to ship?
- [ ] Does `generateFiles` and `staticSource` produce identical output, or are there behavioral differences?

---

## 9. References

- Official docs: <https://www.fumadocs.dev/docs/integrations/openapi>
- `createOpenAPI` server: <https://www.fumadocs.dev/docs/integrations/openapi/server>
- `createOpenAPIPage` UI: <https://www.fumadocs.dev/docs/integrations/openapi/api-page>
- `generateFiles`: <https://www.fumadocs.dev/docs/integrations/openapi/generate-files>
- Redocly CLI (validation): <https://redocly.com/docs/cli/>
- Sibling issue: #11 (API reference) — this doc is the implementation reference
- Sibling issue: #10 (i18n) — per-locale OpenAPI
- Sibling issue: #9 (versioning) — per-version OpenAPI schemas
