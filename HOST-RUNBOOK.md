# HOST-RUNBOOK — arco-review-artifacts

Public, immutable review-artifact host for the Universal Mockup Control Plane.
Served as a static site by GitHub Pages at
`https://arcominion.github.io/arco-review-artifacts/`.

## What this host IS / IS NOT

- **IS** a dumb output surface: static files only, deployed verbatim by GitHub
  Pages. No application, no build step, no server code, no database.
- **IS** append-only and immutable: artifacts live under
  `artifacts/<product>/<issue>/<version>/` and are never edited or overwritten in
  place. A new review is a new `version/` directory.
- **IS NOT** a control plane. It holds no credentials, tokens, or secrets, and it
  **never proxies or brokers credentials**. All authoring/decisioning happens in
  mission-control; this repo only receives finished, sanitized bundles.
- **IS NOT** search-indexable: `robots.txt` disallows all, `.nojekyll` disables
  Jekyll (files served verbatim), and the root `index.html` is a `noindex` + CSP
  confidentiality shell with directory listing disabled.

Anything requiring logic, auth, or freshness belongs upstream in mission-control —
not here. Keep this host dumb.

## How artifacts arrive

Publishing is a two-stage pipeline owned by **mission-control** (this repo runs
none of it):

1. **Publish** — `publishReviewArtifact()` (mission-control
   `src/lib/review-artifacts.ts`, driven by
   `scripts/review-artifact-publish.ts`) writes an immutable bundle into a
   checkout of this host. It validates the profile (product/audience/mode enums,
   git source SHA), requires desktop (≥1200px) **and** true-390 viewport proof,
   enforces an entrypoint-HTML + file-type allowlist (no symlinks, ≤15 MiB),
   rejects secret / production-data markers, re-wraps the entrypoint in the
   confidentiality shell, computes `content_sha256`, and writes `manifest.json`
   (`schema: arco.review-artifact/v1`). It refuses to touch an existing version
   dir — immutability is enforced at write time (`wx`).
2. **Push + readback** — `scripts/review-artifact-host-push.ts` takes a
   single-flight `.host-push.lock` (O_EXCL; stolen only if stale), `git fetch` +
   `merge --ff-only origin/main` (**fail-closed** on divergence — never
   clobbers), stages only the immutable artifact dir, commits
   (`review-artifact <product>/<issue>/<version> (request <id>)`), pushes
   `origin main`, then **polls the public Pages URL until HTTP 200 + marker**
   (or content sha) within `--timeout`. The push to `main` is the deploy trigger.

**Deploy:** `.github/workflows/pages.yml` deploys on every push to `main`,
uploading the repo root (`path: .`) — no build. This is the only automation in
the repo; do not add build steps or change its semantics.

## Retention / expiry

- `mode: expiring` requires a **future** `expiresAt`. The confidentiality shell
  injects a client-side script that **self-blanks** the document once the expiry
  passes — the bytes remain on the immutable host but render nothing useful.
- `archiveExpiredArtifacts(hostRoot)` stamps `archived_at` onto expired
  manifests. `review-artifact-host-push --retention` runs the sweep and commits
  **tracked-only** (`git add -u`, so the held lock is never staged) before
  pushing. Sweeps never delete artifact bytes; the host stays append-only.

## Confidentiality boundary

- **Host-wide:** `robots.txt` Disallow `/`; root `index.html` = `noindex` +
  strict CSP shell; `.nojekyll`.
- **Per-artifact:** every entrypoint is re-shelled with
  `noindex,nofollow,noarchive,nosnippet`, a strict CSP (`connect-src 'none'` — no
  network egress; `object-src`/`base-uri`/`form-action` locked), `no-referrer`,
  and a visible confidentiality banner (`label · issue · version · expires`).
- **Modes:**
  - `public` — link-shareable; still noindex/CSP.
  - `link-only` — share the unguessable URL only (default for external teams).
  - `protected` — client-side AES-256-GCM; viewer enters a separately-shared key;
    the host stores neither key nor plaintext.
  - `expiring` — self-blanks client-side after expiry; server sweep stamps
    `archived_at` (see above).

## Canary artifacts

These are intentionally-retained synthetic proofs (no product content), left in
place because the host is immutable and they are confidentiality-shelled:

- `artifacts/mission-control/umcp-host-canary/v1-eb739d0c/` — UMCP public-host
  live end-to-end publish proof (link-only, external-team, marker
  `UMCP-HOST-CANARY-20260724`).
- `artifacts/saunature-monorepo/sau-canary/v1-cbb2a115/`,
  `.../v2-cbb2a115/` — earlier SAU host canaries.
