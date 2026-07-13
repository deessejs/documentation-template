# Contributing to documentation-template

Thanks for taking the time to contribute. Keep PRs focused — one logical change per PR — and fill out the [pull request template](.github/PULL_REQUEST_TEMPLATE.md).

Please be respectful and follow our [Code of Conduct](CODE_OF_CONDUCT.md).

## Quick Start

```bash
git clone https://github.com/deessejs/documentation-template.git
cd documentation-template
pnpm install        # requires pnpm 10 and Node 22
pnpm dev            # http://localhost:3000
```

## Workflow

1. Branch from `main`: `git switch -c feat/my-change`.
2. Make your change. Keep commits focused.
3. Use [Conventional Commits](https://www.conventionalcommits.org/) for the subject line: `feat:`, `fix:`, `docs:`, `ci:`, `chore:`, `refactor:`, `style:`, `test:`.
4. Push your branch and open a Pull Request using the template. Reference any related issue with `Closes #123`.
5. Make sure all CI checks (`Lint`, `Type Check`, `Build`) pass.

## Conventions

- **MDX content** lives in `content/docs/`, filename in `kebab-case`.
- **Frontmatter** must match the schema declared in `source.config.ts` (`pageSchema` from `fumadocs-core/source/schema`).
- **Sidebar / navigation**: add an entry to `content/docs/meta.json` whenever you add or move a page.
- **Code style**: ESLint flat config (`eslint.config.mjs`) is the source of truth. Do not override per file.
- **Type safety**: `pnpm types:check` must pass.
- **Build**: `pnpm build` must pass — this is a template, broken builds break adopters.
- **No tests** in this repository by design — favor example content over test files.

## Project Structure

```
content/docs/              ← MDX content
src/app/                   ← Next.js routes (home, docs, api, og, llms.*)
src/lib/                   ← shared config, source loader
src/components/            ← MDX component overrides
.github/workflows/         ← CI (lint, types, build)
.github/ISSUE_TEMPLATE/    ← issue forms
```

Do not edit `.source/` — it is regenerated on `pnpm install` via the `fumadocs-mdx` postinstall hook.

## Adding Content

The most common contribution. To add a new documentation page:

1. Create `content/docs/<your-page>.mdx` with frontmatter (`title`, `description`).
2. Add an entry to `content/docs/meta.json` so it appears in the sidebar.
3. Run `pnpm dev` and navigate to `http://localhost:3000/docs/<your-page>`.
4. (Optional) Check `http://localhost:3000/llms-full.txt` to confirm the page is included for LLM consumers.
5. (Optional) Check `http://localhost:3000/og/docs/<your-page>/image.png` for the Open Graph preview.

To change navigation labels or page order, edit `meta.json`.

## Communication

- 🐛 **Bug** → [bug report issue form](.github/ISSUE_TEMPLATE/bug_report.yml)
- ✨ **Feature** → [feature request form](.github/ISSUE_TEMPLATE/feature_request.yml)
- 📚 **Documentation issue** → [docs issue form](.github/ISSUE_TEMPLATE/documentation.yml)
- 💬 **Usage questions** → GitHub Discussions
- 🔒 **Security issue** → see [SECURITY.md](SECURITY.md), do **not** open a public issue

## CI & Merge

- All checks (`Lint`, `Type Check`, `Build`) must pass before review.
- At least one approval is required to merge.
- Squash-merge is preferred to keep the `main` history linear.

## Recognition

Contributors are listed automatically on the GitHub contributors page.
