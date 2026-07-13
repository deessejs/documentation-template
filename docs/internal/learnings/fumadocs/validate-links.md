# Fumadocs Validate Links

> Learning notes derived from the [official Fumadocs docs](https://www.fumadocs.dev/docs/integrations/validate-links). Captured for the `documentation-template` project; complements issue #12 (CI link checker with lychee).

---

## 1. Overview

Fumadocs ships a link validation integration via **`next-validate-link`** — a separate package that scans your MDX content files for broken links and reports them at build/dev time.

This is **distinct from** a CI-only link checker (e.g. `lycheeverse/lychee-action`):

| Aspect | `next-validate-link` (Fumadocs) | `lychee` (CI) |
|---|---|---|
| When it runs | Locally via script, on demand | In CI on every PR/push |
| What it checks | Internal MDX links + headings + component `href` props | All URLs reachable via HTTP |
| External URLs | Skipped by default (HTTP cost) | Validated by HEAD request |
| Speed | Fast (no network) | Slower (network-bound) |
| Best for | Catch link rot in **content** during authoring | Catch link rot in **all** docs after merge |

**Recommendation**: use **both**. `next-validate-link` for fast authoring-time feedback, `lychee` (or equivalent) in CI for full coverage.

---

## 2. Setup — the validation script

Create a script at `scripts/lint.ts` (or `scripts/validate-links.ts`):

```ts
import { type FileObject, printErrors, scanURLs, validateFiles } from 'next-validate-link';
import { source } from '@/lib/source';

async function checkLinks() {
  const scanned = await scanURLs({
    // pick a preset for your React framework
    preset: 'next',
    populate: {
      'docs/[[...slug]]': source.getPages().map((page) => {
        return {
          value: {
            slug: page.slugs,
          },
          hashes: getHeadings(page),
        };
      }),
    },
  });

  printErrors(
    await validateFiles(await getFiles(), {
      scanned,
      // check `href` attributes in different MDX components
      markdown: {
        components: {
          Card: { attributes: ['href'] },
        },
      },
      // check relative paths
      checkRelativePaths: 'as-url',
    }),
    true,
  );
}

function getHeadings({ data }: (typeof source)['inferPage']): string[] {
  return data.toc.map((item) => item.url.slice(1));
}

function getFiles() {
  const promises = source.getPages().map(async (page): Promise<FileObject> => ({
    path: page.absolutePath,
    content: await page.data.getText('raw'),
    url: page.url,
    data: page.data,
  }));

  return Promise.all(promises);
}

void checkLinks();
```

> Note: the docs use `(typeof source)['$inferPage']`. In current `next-validate-link` versions the type accessor may be named `inferPage` instead of `$inferPage` — verify against the installed version.

---

## 3. How it works — the three phases

### Phase 1 — `scanURLs`

Builds a map of all **valid URLs** the docs site can produce:

- Resolves dynamic routes (`docs/[[...slug]]`) by enumerating `source.getPages()`.
- For each page, captures:
  - The slug values that produce a valid URL.
  - The heading anchors available on that page (from `page.data.toc`).

The output (`scanned`) is a per-route descriptor the validator uses as ground truth.

### Phase 2 — `validateFiles`

Reads every MDX file via `page.data.getText('raw')` and checks every link/anchor/component prop:

- **Internal links**: `/docs/some-page` or `[[...slug]]`-resolved paths.
- **Anchors**: `#some-heading` — must exist in the target page's `toc`.
- **Component props**: MDX components with `href`-like attributes (customizable via `markdown.components`).
- **Relative paths**: `./foo.mdx`, `../bar.mdx` — resolved per `checkRelativePaths` mode.

### Phase 3 — `printErrors`

Outputs a human-readable report. The second argument `true` makes it exit non-zero on errors (CI-friendly).

---

## 4. Configuration options

### `scanURLs` options

| Option | Purpose |
|---|---|
| `preset` | Framework adapter (`'next'`, `'react-router'`, etc.) — defines URL conventions |
| `populate` | Map of route patterns to lists of dynamic values + heading hashes |

### `validateFiles` options

| Option | Default | Purpose |
|---|---|---|
| `markdown.components` | `{}` | Tell the validator which MDX components have `href`-like attributes |
| `checkRelativePaths` | `'as-url'` | How to resolve relative `.mdx` paths. `'as-url'` treats them as routes; `'fs'` treats them as filesystem paths |

For `documentation-template`, the `Card` component entry is essential because that's how the existing example pages link to other docs:

```tsx
<Cards>
  <Card title="Learn more about Next.js" href="https://nextjs.org/docs" />
  <Card title="Learn more about Fumadocs" href="https://fumadocs.dev" />
</Cards>
```

If you have additional linking components (e.g. a custom `LinkCard`, `ExternalLink`, `DownloadLink`), add them to the `components` map.

---

## 5. Running the validation

The Fumadocs docs recommend running the script via **Bun**:

```bash
bun ./scripts/lint.ts
```

### Why Bun?

The script imports `@/lib/source`, which is an alias defined in `tsconfig.json`. Node's experimental TS support doesn't honor `tsconfig.json` paths. Bun does.

### Alternatives

1. **Bun** (recommended by Fumadocs) — `bun ./scripts/lint.ts`
2. **tsx** (Node-compatible, reads `tsconfig.json` paths) — `pnpm dlx tsx ./scripts/lint.ts`
3. **Node 22+ with `--experimental-strip-types`** — `node --experimental-strip-types ./scripts/lint.ts` (limited)
4. **Configure Fumadocs MDX Loader for Node** — see "Node.js Loader" section in the docs; more setup

For `documentation-template`, **`tsx` is the natural choice** since the project already uses pnpm and TypeScript paths.

Add to `package.json`:

```json
{
  "scripts": {
    "lint:links": "tsx scripts/lint.ts"
  }
}
```

Then:

```bash
pnpm lint:links
```

---

## 6. NPM script wiring

Update `package.json`:

```json
{
  "scripts": {
    "lint": "eslint",
    "lint:links": "tsx scripts/lint.ts",
    "lint:all": "pnpm lint && pnpm lint:links"
  }
}
```

`lint:all` becomes the single command for pre-commit hooks and the CI workflow.

---

## 7. Integration with CI

The script exits non-zero on broken links (`printErrors(..., true)`). Wire it into the existing `lint.yaml` workflow:

```yaml
name: Lint

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: pnpm/action-setup@v6
        with:
          version: 10
      - uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: 'pnpm'
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      - name: Run ESLint
        run: pnpm lint
      - name: Validate MDX links
        run: pnpm lint:links
```

This **complements** the lychee workflow from issue #12:
- `next-validate-link` runs in **Lint** — fast, content-focused.
- `lychee` (if added) runs in **link-checker.yaml** — slow, external-URL focused.

---

## 8. Known caveats

### `next-validate-link` limitations

- **External URLs are NOT validated by default** — would require HTTP requests per link (expensive).
- **Component prop validation is opt-in** — you must declare every component that has link-like props.
- **Hash validation depends on `toc`** — if a heading is generated dynamically (e.g. via MDX components), it won't be detected.

### When the validator misses things

- Links inside MDX expressions like `{someVar}` (dynamic hrefs) are skipped.
- Links generated by `<DynamicLink href="/[lang]/...">` (i18n) may need special handling — see the i18n learning doc.
- Components imported from MDX but not statically rendered (e.g. inside `Tabs`/`Tab` boundaries) may not be detected depending on parser depth.

### Versioning interaction

If/when issue #9 (versioning) lands, `scanURLs` needs to be **per-version**:

```ts
populate: {
  'docs/v2/[[...slug]]': source.getPages('v2').map(...),
  'docs/v1/[[...slug]]': source.getPages('v1').map(...),
}
```

Links between versions (`/docs/v1/foo` → `/docs/v2/bar`) need to be cross-checked.

### i18n interaction

If/when issue #10 (i18n) lands, `scanURLs` needs to be **per-locale**:

```ts
populate: {
  '(en)/docs/[[...slug]]': source.getPages('en').map(...),
  '(fr)/docs/[[...slug]]': source.getPages('fr').map(...),
}
```

The route pattern changes from `docs/[[...slug]]` to `[lang]/docs/[[...slug]]`.

---

## 9. Comparison with lychee

| Dimension | `next-validate-link` | `lychee` |
|---|---|---|
| **Where** | Local + CI (lint step) | CI only |
| **Speed** | Seconds | Minutes (network-bound) |
| **Cost** | Free | Free, but rate-limited by external sites |
| **External URLs** | No | Yes |
| **Anchor validation** | Yes (via `toc`) | No |
| **Component props** | Yes (configurable) | No (plain HTML only) |
| **MDX-aware** | Yes | No (treats as text) |
| **False positives** | Low | Medium (rate limits, blocks) |
| **Setup cost** | Low (one script) | Low (one workflow) |
| **Maintenance** | Update `components` map as MDX evolves | Add `exclude` patterns as docs grow |

### Recommendation for `documentation-template`

1. **Add `next-validate-link`** as the primary lint tool (fast, content-aware).
2. **Add `lychee` later** as the secondary CI guard (slow, external URL coverage).
3. **Document both** in `CONTRIBUTING.md` under a "Validating links" section.

---

## 10. Specific considerations for `documentation-template`

### What ships in this template today

The example MDX files use:
- `<Cards>` with `<Card href="https://...">` — external links
- No internal cross-references between MDX pages yet

The validator config needs at minimum:

```ts
markdown: {
  components: {
    Card: { attributes: ['href'] },
  },
},
```

### When the template grows

As adopters add custom components (per #8 shadcn support), they should add corresponding entries:

```ts
markdown: {
  components: {
    Card: { attributes: ['href'] },
    Button: { attributes: ['href'] },
    LinkCard: { attributes: ['href'] },
    ExternalLink: { attributes: ['href'] },
    // ...adopter-specific
  },
},
```

Document this in `CONTRIBUTING.md` so adopters know to extend the validator map.

### Estimated effort to integrate

- **New file**: `scripts/lint.ts` (~30 lines)
- **Modified files**:
  - `package.json` — add `lint:links` script + `next-validate-link` devDependency
  - `.github/workflows/lint.yaml` — add the `pnpm lint:links` step
  - `CONTRIBUTING.md` — document the new script
- **Effort**: ~30 minutes for a contributor familiar with the repo

---

## 11. Open questions

- [ ] Which version of `next-validate-link` is compatible with the current Fumadocs and Next.js 16?
- [ ] Does `tsx` correctly resolve `@/lib/source` from `scripts/lint.ts` (one folder above `src/`)?
- [ ] Should the validator ignore example URLs in starter content (like the Fumadocs links in `index.mdx`)? They may rot.
- [ ] Does `validateFiles` support MDX expressions, or only static attributes?
- [ ] How does the validator handle Fumadocs' `<DynamicLink href="/[lang]/...">` — does it substitute `[lang]` before checking?

---

## 12. References

- Official docs: <https://www.fumadocs.dev/docs/integrations/validate-links>
- `next-validate-link` package: <https://github.com/fuma-nama/next-validate-link>
- Sibling CI approach (lychee): issue #12 in this repo
- i18n interaction: `docs/internal/learnings/fumadocs/internationalization.md`
- Versioning interaction: issue #9 in this repo
