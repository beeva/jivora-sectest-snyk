# Jivora — security platform test bed · Snyk

> ⚠️ **Intentionally vulnerable.** This repository exists solely to evaluate
> security scanning platforms. Do not deploy it, do not reuse its code, and do
> not treat any credential in it as real.

**Platform under test:** [Snyk](https://snyk.io) — Snyk Open Source (SCA),
Snyk Code (SAST), Snyk IaC and Snyk Container.

## Enabling it

1. Sign up at [snyk.io](https://snyk.io) and import this repository through the
   GitHub integration. That alone gives you Open Source and IaC scanning plus
   automatic fix pull requests.
2. **Snyk Code is off by default.** Turn it on under
   *Settings → Snyk Code* in the organisation settings, or you will silently
   evaluate Snyk with its SAST engine disabled.
3. For the CI arm, add a `SNYK_TOKEN` repository secret. The workflow in
   `.github/workflows/snyk.yml` runs all four product lines and uploads SARIF to
   the GitHub Security tab, so results sit next to every other tool's.

## What to watch for

Snyk's strongest arm is **Open Source**, and this repository is built to stretch
it:

- **Reachability.** 25 dependencies are pinned to versions with known
  advisories, but only some are genuinely reachable. `lodash` is called
  directly by the prototype-pollution sink; `axios` by the SSRF; `marked` by the
  XSS. Others are transitive noise. Snyk's reachability analysis is a headline
  feature — check whether it actually separates the two.
- **Fix PRs.** Snyk will open upgrade pull requests. Judge them: does it batch
  sensibly across three `package.json` files, does it propose a `next` upgrade
  that would break the App Router, and what does it do with `xlsx`, which has no
  fixed version on the public npm registry?
- **Yarn vs npm.** `jivora-admin` is Yarn-only by design. Watch whether Snyk
  handles the mixed-package-manager monorepo or quietly scans two of three apps.

For the other arms:

- **Snyk Code** should find the injection and traversal findings. Compare its
  depth against CodeQL on the same code — this is the most direct SAST
  comparison in the bake-off, since both run over identical sources.
- **Snyk IaC** covers Terraform, Kubernetes and Compose. 19 findings are waiting
  in `infra/`, plus 8 in `docker-compose.yml`.
- **Snyk Container** needs the images built first; the workflow does that.
  Expect heavy base-image noise from `node:18` — that noise is itself a data
  point about triage burden.
- **Blind spot to confirm:** Snyk's secret detection is thin compared to the
  rest of the field. The 8 findings in category H are the test.

## Cost note

Snyk's free tier is generous on test count but limits Snyk Code and the
higher-value features. The per-developer pricing escalates sharply above the
team tier — model it at 10 developers before comparing against Aikido, because
that is where the two diverge.
---

## What this repository is

A deliberately vulnerable, structurally faithful miniature of **Jivora** — a
Lyme-disease wellness platform handling PHI under HIPAA. It exists to be scanned.

The shape is copied from the real system's architecture document (v1.4), so
findings land on the same surfaces they would in production: a monorepo of three
applications over one PostgreSQL database, two API surfaces with different
authentication, PHI column encryption, an AI assistant with a read-only tool
loop, Stripe webhooks, and a Docker Compose deployment.

```
Jivora-backend/     Node 22 · Express 4 · Sequelize · Socket.IO · @anthropic-ai/sdk
jivora-web/         Next.js App Router · member app · port 4000
jivora-admin/       Next.js App Router · admin panel · port 4002 · Yarn only
infra/terraform/    The AWS deployment the platform would move to
infra/k8s/          The Kubernetes topology for the same
docker-compose.yml  The production stack as actually deployed today
.env.prod           Committed on purpose
```

### Architectural fidelity

| Real Jivora | Here |
| --- | --- |
| Two account stores (`user`, `admin_user`), seven roles | Both, with the same role ids |
| `/api/web/*` and `/api/*` registered separately | Both registrars present |
| PHI column encryption + `*_hash` lookup columns | Present — and subtly wrong |
| Entitlements recomputed live, 402 on paywall | `lib/web/entitlements.js` |
| Admin route manifest as single source of truth | `lib/admin/route-manifest.js` + `route-access.js` |
| 13 read-only assistant tools bound to `user.id` | All 13 — one is not bound |
| Stripe webhook signature mandatory | Claimed in docs, not in code |
| In-process scheduler, no separate worker | `lib/web-protocol/reminder-scheduler.js` |

Where the code contradicts the architecture document, that is the point: the
most interesting findings in this repository are the ones where a stated
security control does not survive contact with the implementation.

## Running it

Don't. There is no seed data, and several dependencies are pinned to versions
with known remote-code-execution advisories. If you want a container for
image-layer scanning:

```sh
docker build -t jivora/backend:sectest Jivora-backend
docker build -t jivora/web:sectest     jivora-web
```

`npm install` inside any of the three apps is enough for dependency scanning and
is safe as long as nothing is executed.

## Scoring a platform

1. Connect the platform. Do not read `FINDINGS.md` first.
2. Let it scan everything it is willing to scan.
3. Export its findings.
4. Now open [`FINDINGS.md`](FINDINGS.md) — the answer key: 57 code and config
   issues across 7 categories, plus 25 dependencies chosen for known advisories.
5. Fill in [`SCORECARD.md`](SCORECARD.md).

Four multi-step attack chains are planted deliberately. Whether a platform
surfaces them *as chains* rather than as unrelated findings is the sharpest
differentiator in this repository.

## Sibling repositories

The same codebase, scanned by a different platform in each:

| Repository | Platform |
| --- | --- |
| [`jivora-sectest-github-advanced-security`](https://github.com/beeva/jivora-sectest-github-advanced-security) | GitHub Advanced Security |
| [`jivora-sectest-aikido`](https://github.com/beeva/jivora-sectest-aikido) | Aikido |
| [`jivora-sectest-snyk`](https://github.com/beeva/jivora-sectest-snyk) | Snyk |
| [`jivora-sectest-veracode`](https://github.com/beeva/jivora-sectest-veracode) | Veracode |
| [`jivora-sectest-wiz`](https://github.com/beeva/jivora-sectest-wiz) | Wiz |

Identical content by design — only this README and the workflows in
`.github/workflows/` differ, so the results are directly comparable.

## Safety

Every credential in this repository is synthetic and authenticates against
nothing. `AKIAIOSFODNN7EXAMPLE` is AWS's own published documentation key. There
is no real PHI here and no real patient data; the clinical vocabulary is
structural, and every value is invented.

GitHub push protection is disabled on this repository on purpose — the committed
`.env.prod` is one of the things being tested for.

Do not copy any pattern from this repository into anything real.
