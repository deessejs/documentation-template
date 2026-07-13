<!--
  Thanks for contributing to deessejs/documentation-template!
  Fill out the sections below. Keep PRs focused — one logical change per PR.
-->

## Description

<!-- What does this PR do, and why? Link any related issue with "Closes #123". -->

Closes #

## Type of change

<!-- Check all that apply. -->

- [ ] 🐛 Bug fix (non-breaking change that fixes an issue)
- [ ] ✨ New feature (non-breaking change that adds functionality)
- [ ] 💥 Breaking change (fix or feature that changes existing behavior)
- [ ] 📚 Documentation only
- [ ] 🧹 Chore / refactor / tooling (no runtime or template-surface change)

## Affected area

<!-- Check all that apply. -->

- [ ] content (MDX files / frontmatter)
- [ ] layout (theme / sidebar / navigation)
- [ ] search (Orama / `/api/search`)
- [ ] llms.txt / llms-full.txt / llms.mdx (content negotiation)
- [ ] og images (`/og/docs/...`)
- [ ] proxy / content negotiation middleware
- [ ] CI workflows (`.github/workflows/`)
- [ ] issue templates / PR template
- [ ] documentation (README / setup instructions)
- [ ] example / showcase (new content file)
- [ ] build / packaging / dependencies

## Checklist

- [ ] `pnpm lint` passes
- [ ] `pnpm types:check` passes
- [ ] `pnpm build` passes
- [ ] I verified the affected pages render correctly with `pnpm dev` (when relevant)
- [ ] I added or updated example content under `content/docs/` to cover the change (when relevant)
- [ ] I updated the README / docs / JSDoc if the template config or public surface changed
- [ ] I updated `content/docs/meta.json` if a new page should appear in the sidebar (when relevant)
- [ ] I did not unintentionally change `src/lib/shared.ts`, `source.config.ts`, or `next.config.mjs` defaults
- [ ] If a dependency was added or upgraded, the change is intentional and reflected in `pnpm-lock.yaml`

## Notes for reviewers

<!-- Anything reviewers should focus on, trade-offs made, or follow-ups deferred. -->
