# Security Policy

If you discover a security vulnerability in this repository, **please do not open a public issue**. Use the private channel below so we can triage and fix it before public disclosure.

## Supported Versions

Only the latest commit on the `main` branch is actively maintained. Older commits are not patched — please repin or rebase before reporting.

## Reporting a Vulnerability

Report privately through [GitHub Security Advisories](https://github.com/deessejs/documentation-template/security/advisories/new).

Please include:

- **Type** of issue (XSS, RCE, info disclosure, supply chain, DoS, …)
- A short **description** and **impact assessment**
- **Steps to reproduce**
- The **affected commit or version**
- Whether it has already been disclosed publicly

## Response Timeline

- **Acknowledgement**: within 72 hours
- **Triage and severity**: within 7 days
- **Fix**: depends on severity (critical → days; low → weeks)
- **Public disclosure**: coordinated, after a fix is available

## Out of Scope

- **Vulnerabilities in upstream dependencies** (Next.js, Fumadocs, Tailwind, lucide-react, …) — please report directly to the affected project.
- **Issues in user forks** of this template — these are the adopter's responsibility.
- **Anything that is not a security issue** — please use the regular [issue templates](.github/ISSUE_TEMPLATE/).

## Hardening Recommendations for Adopters

When you fork this template to host your own docs, consider enabling:

- **Dependabot security updates** — add `.github/dependabot.yml` with `package-ecosystem: npm` (or `pnpm`) and a sensible schedule.
- **GitHub CodeQL** — free on public repos, under the *Security* tab.
- **Secret scanning + push protection** — GitHub repository settings.
- **Security headers** — extend `next.config.mjs` with `headers()` for `Content-Security-Policy`, `Strict-Transport-Security`, `X-Frame-Options`, `Referrer-Policy`, and `Permissions-Policy`.
- **Lockfile pinning in CI** — already enforced here via `pnpm install --frozen-lockfile` in the lint, types, and build workflows.
- **A private package proxy** if you publish internal packages consumed by your fork.

These are not required to use the template, but they significantly reduce the attack surface of any deployed documentation site.
