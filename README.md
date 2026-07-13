# Documentation Template

> Production-ready documentation starter built with Next.js 16, Fumadocs, and Tailwind v4.

[![Build](https://github.com/deessejs/documentation-template/actions/workflows/build.yaml/badge.svg?branch=main&event=push)](https://github.com/deessejs/documentation-template/actions/workflows/build.yaml)
[![Type Check](https://github.com/deessejs/documentation-template/actions/workflows/types.yaml/badge.svg?branch=main&event=push)](https://github.com/deessejs/documentation-template/actions/workflows/types.yaml)
[![Lint](https://github.com/deessejs/documentation-template/actions/workflows/lint.yaml/badge.svg?branch=main&event=push)](https://github.com/deessejs/documentation-template/actions/workflows/lint.yaml)

A complete documentation site out of the box: write MDX, get full-text search, auto-generated OG images, LLM-friendly endpoints, content negotiation, and CI.

## Quick start

```bash
git clone https://github.com/deessejs/documentation-template.git my-docs
cd my-docs
pnpm install
pnpm dev
```

Open <http://localhost:3000>.

## Use as a template

Click **Use this template** on GitHub to scaffold a new repository, or [fork](https://github.com/deessejs/documentation-template/fork) this one directly.

## Customizing

- App name and GitHub link — `src/lib/shared.ts`
- Content — add `.mdx` files in `content/docs/` and reference them from `content/docs/meta.json`
- Layout, theme, navigation — `src/lib/layout.shared.tsx`
- Routes — `src/app/`

See the local docs at [`/docs`](http://localhost:3000/docs) for a working example.

## Project structure

```
content/docs/             MDX content
src/app/                  Next.js routes (home, docs, api, og, llms.*)
src/lib/                  shared config, source loader
src/components/           MDX component overrides
.github/workflows/        CI (lint, types, build)
```

## Scripts

| Script             | Purpose                                       |
| ------------------ | --------------------------------------------- |
| `pnpm dev`         | Start dev server                              |
| `pnpm build`       | Production build                              |
| `pnpm start`       | Serve production build                        |
| `pnpm lint`        | Run ESLint                                    |
| `pnpm types:check` | Regenerate `.source/` and run `tsc --noEmit`  |

## Routes

| Route                     | Description                                |
| ------------------------- | ------------------------------------------ |
| `/`                       | Landing page                               |
| `/docs`                   | Documentation site                         |
| `/api/search`             | Full-text search (Orama)                   |
| `/og/docs/[...slug]`      | Generated Open Graph images                |
| `/llms.txt`               | LLM-friendly sitemap index                 |
| `/llms-full.txt`          | Concatenated LLM-readable content          |
| `/llms.mdx/[...slug]`     | Markdown source for individual pages       |

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md). Open issues using the [bug](.github/ISSUE_TEMPLATE/bug_report.yml), [feature](.github/ISSUE_TEMPLATE/feature_request.yml), or [documentation](.github/ISSUE_TEMPLATE/documentation.yml) templates.

## Security

Report private vulnerabilities per [SECURITY.md](SECURITY.md) — do not open public issues.

## License

[MIT](LICENSE).
