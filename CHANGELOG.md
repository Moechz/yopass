# Changelog

All notable user-visible changes to the Yopass TOS application package.

## 16.2.0-1 — 2026-11-21

Remediation release for the App Center listing review of 2026-11-20
(`C10`, `C3`, `C9`, `T1`, plus the App Center self-test installation failure).
"Rectified per review comments."

### Security (T1)
- Upgraded the image from `jhaals/yopass:16.1.0` to **`jhaals/yopass:16.2.0`**.
  16.2.0 fixes the high-severity recipient-side key-derivation issue
  **GHSA-4v9j-h5v5-4vjw** (a crafted secret could make the recipient's browser
  spend excessive time in Argon2 before failing). The affected code runs in the
  web client shipped in this image; the server never decrypts.
- 16.2.0 makes the Rust server the default for the `16.2.0`/`latest` tags. It
  accepts the same flags, environment variables and data formats as the Go
  server, so **no data migration is required**. The `/yopass` CLI is no longer
  bundled in the image; this package never used it. OIDC session cookies (not
  enabled here) would require signing in again.
- Migration note for reviewers: see `docs/REVIEW-REMEDIATION-2026-11.md`.
  Pending memcached secrets are lost when the containers are recreated, which is
  expected for the in-memory cache; files under `data/files` are unaffected.

### Privacy policy (C3)
- Added `PRIVACY.md`: end-to-end encryption and URL-fragment key handling,
  ciphertext-only server storage (memcached and `data/files`), retention and
  one-time deletion, logs, user rights/deletion, security measures, and the
  explicit statement that the app has no accounts, analytics or telemetry.
- `config.ini` `help` and `tosapp.json` `help` now point to
  <https://github.com/Moechz/yopass/blob/main/PRIVACY.md>.

### Description consistency (C9)
- Rewrote the `zh-cn` and `zh-hk` descriptions, which contained text from
  another application (habit tracking) instead of Yopass.
- Updated every language's `descript`/`important` text to match the actual
  functionality: browser-side encryption, one-time (single-view) links,
  1 hour / 1 day / 1 week expiry, optional custom password, and ciphertext-only
  storage. No arbitrary "max views" setting is advertised because the app does
  not have one.

### Repository and README compliance (C10)
- README now documents the **Repository layout** and a **Privacy** section, and
  describes the two-service compose stack, port 8180 and the data layout.
- Removed stale build-probe metadata (`probe`, `entrypoint_file`) that described
  a different application from `tosapp.json`; the remaining fields match this
  package.

### App Center installation failure
- Moved the host port from **8142 to 8180**. Port 8142 is already registered by
  another catalog app (`nzbhydradocker`), which can make installation fail or
  open the wrong service. `config.ini`, `docker-compose.yml`, `x-app-meta`,
  README and the privacy policy are consistent on 8180.
- Switched `depends_on` to the short compose form for compatibility with the
  platform installer, and made the app health check use Yopass's
  `--health-check` probe (verifies the memcached connection) instead of
  `--help`. The `MEMCACHED` environment variable is set because the
  health-check process does not inherit the container command flags.

## 16.1.0-1 — 2026-11-14

### Added
- First release. Yopass (`jhaals/yopass:16.1.0`, Docker Hub official image, unmodified)
  packaged as a TOS 7 Docker application, published on port 8142 and running
  as a non-root user (1000:1000).
- Persistent data is kept under `/Volume1/DockerAppData/yopassdocker/`.
