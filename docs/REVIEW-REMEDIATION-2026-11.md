# Review remediation — App Center listing review 2026-11-20

- **App:** Yopass (`yopassdocker`), TerraMaster TOS 7
- **Reviewed submission:** `16.1.0-2` (x86_64)
- **Remediated submission:** `16.2.0-1` (x86_64 + aarch64)
- **Review codes:** C10, C3, C9, T1 — plus the App Center self-test installation
  failure reported in the same notice.
- **Submission remark:** "Rectified per review comments."

## Mapping

| Code | Review finding | What changed | Where to verify |
|---|---|---|---|
| C10 | Repository and directory compliance: make the source repository public, complete the README, align the directory layout | Repository is public. README now has **Repository layout**, **Privacy**, build and submission sections. Removed stale `probe`/`entrypoint_file` metadata that described a different application. | `README.md`, `tosapp.json`, this document |
| C3 | Privacy policy missing | Added `PRIVACY.md` covering E2E encryption, URL-fragment keys, ciphertext-only storage, retention/one-time deletion, logs, user rights/deletion, security measures, no accounts/telemetry. `config.ini` `help` and `tosapp.json` `help` point to the public policy URL. | `PRIVACY.md`, `src/config.ini`, `tosapp.json` |
| C9 | Description inconsistent with delivered functionality | Rewrote `zh-cn`/`zh-hk`, which contained text from another app (habit tracking). All languages now describe the real features: browser-side encryption, one-time/single-view links, 1 hour/1 day/1 week expiry, optional custom password. No arbitrary "max views" claim. | `src/yopassdocker.lang`, `tosapp.json.translations` |
| T1 | High/critical CVE | Upgraded `jhaals/yopass:16.1.0` → `jhaals/yopass:16.2.0`, which fixes GHSA-4v9j-h5v5-4vjw (high severity, recipient-side Argon2 key-derivation cost). | `src/docker-compose.yml`, `tosapp.json`, `CHANGELOG.md` |
| App Center self-test | Online installation failed — app cannot be opened | Host port 8142 collided with the published catalog app `nzbhydradocker` (also 8142). Moved to **8180** everywhere. Also switched `depends_on` to the short compose form and changed the app health check to `--health-check` so it verifies the memcached connection. | `src/config.ini`, `src/docker-compose.yml`, `README.md` |

## T1 detail and migration note

- **Issue:** GHSA-4v9j-h5v5-4vjw. The sender chooses the key-derivation cost of a
  secret; a crafted secret could keep the **recipient's browser or CLI** busy
  for seconds to hours. The server never decrypts, so servers were not affected.
- **Fix:** Yopass 16.2.0 refuses messages with more than 16 Argon2 passes,
  parallelism above 16, or more than 8 password key packets before deriving any
  key. The web client is included in the upgraded image.
- **Image:** `jhaals/yopass:16.2.0`, used unmodified from Docker Hub. It is a
  multi-arch image (`linux/amd64`, `linux/arm64`).
- **Migration:** none for data. 16.2.0 promotes the Rust server to the version
  tags; it takes the same flags and environment variables and reads/writes the
  same data as the Go server.
  - Pending secrets in memcached are lost when the containers are recreated.
    This is normal for the in-memory store; uploaded files under `data/files`
    persist.
  - The `/yopass` CLI is no longer included in the image; this package does not
    use it.
  - If OIDC authentication were enabled (it is not in this package), users would
    have to sign in again because session-cookie formats changed.
- **Next review:** the CVE remediation should be verified against the 16.2.0
  image; no 16.1.0 bits remain in the compose file.

## Installation failure — root cause and fix

The review notice also reported: **"App Center test online installation failed —
cannot be opened."** The package itself passes its self-test; the install path
failed at runtime. The concrete packaging defect found was a host-port
collision:

- `yopassdocker` used host port **8142**;
- the published catalog app **`nzbhydradocker`** (NZBHydra 2, x86_64 and
  aarch64) also registers host port **8142** in the catalog registry.

On a NAS with NZBHydra 2 installed, the new container cannot bind 8142 and the
App Center cannot open the app. The fix moves Yopass to **8180**, which is free
in the catalog registry and outside TOS's reserved ports.

Additional hardening for the install path:

- `depends_on` uses the short compose list form, which the platform installer
  handles consistently (the long `condition: service_healthy` form is valid
  Compose but is not needed here).
- The app health check is now `/yopass-server --health-check`, which verifies
  the memcached connection, instead of `--help`, which only printed usage. The
  `MEMCACHED` environment variable is set so the separate health-check process
  (which does not inherit the command flags) reaches the internal cache.
- The app and memcached still run as `1000:1000`; the only published port is
  8180.

## Verification for this submission

| Gate | Result |
|---|---|
| `scripts/build-dual.sh 16.2.0-1` (self-check + official `-c`, both arches) | see build log below |
| `python3 stage.py work/yopass --timeout 300` | see staging log below |
| Anonymous release asset reachability | checked by `ghops.py publish` |

### Build

```text
== build yopassdocker 16.2.0-1 (aarch64)
verify OK: files=['config.ini', 'docker-compose.yml', 'yopassdocker.lang',
' yopassdocker.svg'] langs=23 icon_elements=15 icon_bytes=3364
host_port=8180 volumes=1 mode=plain
== build yopassdocker 16.2.0-1 (x86_64, canonical)
verify OK: files=['config.ini', 'docker-compose.yml', 'yopassdocker.lang',
' yopassdocker.svg'] langs=23 icon_elements=15 icon_bytes=3364
host_port=8180 volumes=1 mode=plain
== official check (both architectures)
check passed
check passed
939e8783dd34c2ac2967f05e8c3b3ac7de0ce55b7d9d194f2d0a1b5d189a45f8  out/yopassdocker_x86_64.tar.gz
fd1ad01cfc6830cc30bf0ffb5f0aafb4d21d7982bfbaa420ce34728f3f92448a  out/yopassdocker_aarch64.tar.gz
```

### Staging

```text
== staging yopassdocker 16.2.0-1 (隔离端口 + /tmp 数据)
   port map: yopassdocker 8180 -> 127.0.0.1:39002 (container 1337)
  [ OK ] 2 个容器 running 且 health 正常
  [ OK ] yopassdocker: 127.0.0.1:39002 -> HTTP 200
  [ OK ] stageyopassdocker-yopassdocker-1: user=1000:1000 (非 root)
  [ OK ] stageyopassdocker-yopassdocker-memcached-1: user=1000:1000 (非 root)
  [staging] passed 4, failed 0  (keep=False)
```

### Functional round-trip

With the staged stack kept, an OpenPGP test message was posted and read back:

```text
POST /create/secret
  -> {"message":"7gnWIvygRZJ8wvZjlmO0qU"}
GET /secret/7gnWIvygRZJ8wvZjlmO0qU
  -> {"expiration":3600,"message":"-----BEGIN PGP MESSAGE-----..."}
GET /secret/7gnWIvygRZJ8wvZjlmO0qU/status
  -> {"oneTime":false,"requireAuth":false}
gpg --decrypt (with the test passphrase)
  -> smoke-test-secret-16.2.0
```

This exercises the app container, the internal memcached service, the health
check and the encrypted store. The staging containers and data were removed
afterwards.
