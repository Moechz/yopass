# yopass-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [Yopass](https://github.com/jhaals/yopass).

## What this is

A four-file TOS Docker application archive (`yopassdocker.tar.gz`) that installs
Yopass on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `yopassdocker`)
- `yopassdocker.lang` — 23-language superset (TOS requires 14)
- `yopassdocker.svg` — application icon (3364 bytes, 5 SVG elements)
- `docker-compose.yml` — two non-root services, image `jhaals/yopass:16.2.0`
  plus an internal `memcached:1.6-alpine` cache

## Repository layout

```text
.
├── README.md                  # this file: what the package is, layout, privacy
├── PRIVACY.md                 # privacy policy (linked from config.ini help)
├── CHANGELOG.md               # user-visible package changes
├── LICENSE                    # Apache-2.0 (upstream)
├── tosapp.json                # packaging spec used by scripts/build.sh
├── docs/
│   └── REVIEW-REMEDIATION-2026-11.md  # C3/C9/C10/T1 + install-failure mapping
├── scripts/
│   ├── build.sh               # single-arch build + verify.py + tar
│   └── build-dual.sh          # x86_64 + aarch64 build and official -c check
├── src/
│   ├── config.ini             # App Center metadata (version/platform templated)
│   ├── docker-compose.yml     # app + memcached, non-root, healthchecks
│   ├── yopassdocker.lang      # 23-language description texts
│   └── yopassdocker.svg       # icon
├── test/install-and-verify.sh # on-device checks (root; not run by CI/nightly)
└── tools/verify.py            # review-standards self-check used by build.sh
```

Only the four files listed under "What this is" are placed inside the published
archive; everything else is repository material for review and rebuilds.

## How it works

| | |
|---|---|
| Container image | `jhaals/yopass:16.2.0` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8180/` (container port 1337) |
| Runs as | non-root, `user: "1000:1000"` (both app and memcached) |
| Persistent data | `/data` → `/Volume1/DockerAppData/yopassdocker/data` |
| Logs | `docker logs yopassdocker` (stdout/stderr) |

Yopass encrypts the secret in the browser; the key (and an optional custom
password) is placed in the URL fragment, which browsers do not send to the
server. The server stores only ciphertext, in memcached while the secret lives
and on disk for uploaded files. Links can be one-time (single view) and expire
after one hour, one day or one week. No account is needed.

## Privacy

See **[PRIVACY.md](PRIVACY.md)** — the policy linked from `config.ini`'s `help`
field and summarised here:

- end-to-end encryption in the browser; the key never reaches the server;
- server-side storage is ciphertext only (memcached + `/data/files`), with
  expiry and one-time deletion;
- no accounts, no analytics, no telemetry, no third-party trackers or CDNs;
- container logs contain operational metadata, never plaintext or keys;
- the NAS administrator controls deletion (link, expiry, or removing the data
  folder).

## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, and no Docker socket, no `cap_add`.
- **Health checks** for both services; the app uses Yopass's own `--health-check`
  probe, which verifies the memcached connection instead of only printing help.
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check above.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `/Volume1/DockerAppData/yopassdocker/data` | Yopass data root (mounted at `/data`) | on install / first start | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |
| `/Volume1/DockerAppData/yopassdocker/data/files` | encrypted uploaded files (`--file-store disk`) | on first file upload | bounded by upload size and lifetime | cleaned automatically after expiry |

Pending text secrets live in the internal memcached container (memory only) and
are lost when the app or memcached container is restarted.

## Build

```bash
scripts/build.sh                     # -> out/yopassdocker.tar.gz + .sha256
scripts/build-dual.sh 16.2.0-1       # x86_64 + aarch64 + official -c
scripts/build.sh 16.2.0-1 aarch64    # single architecture
```

The build stages the four files, substitutes the version/platform placeholders,
removes CRLF/BOM/AppleDouble, then runs
`tools/verify.py`, a review-standards self-check (archive layout, JSON validity,
required fields, 14+ language coverage, version consistency, Docker-Hub-only
images, fixed tag, reserved/recommended host ports, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, never the
`/Volume*` wildcard, comment-free sequence blocks, `x-app-meta` position, no
literal secrets and the SVG size/element limits).

## Local verification

```bash
./TOSAppSelfTestingTool -c out/yopassdocker.tar.gz   # official spec check
bash test/install-and-verify.sh                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README + PRIVACY).
2. Release **tag `v16.2.0-1`**, assets `yopassdocker_x86_64.tar.gz`,
   `yopassdocker_aarch64.tar.gz` and their `.sha256` files.
3. Developer platform → app `yopassdocker`, package type Docker, repository URL,
   architecture `x86_64` and `aarch64`.
4. Version Management → Add Version `16.2.0-1` → automated validation → review.

## Changes in 16.2.0-1

This release remediates the App Center review findings C3, C9, C10 and T1, and
the App Center self-test installation failure:

- **T1** — upgraded the upstream image from 16.1.0 to **16.2.0**, which fixes
  the high-severity recipient-side Argon2 key-derivation issue
  (GHSA-4v9j-h5v5-4vjw). The 16.2.0 server is the Rust implementation; it accepts
  the same flags and data, so no migration is needed (see
  [`docs/REVIEW-REMEDIATION-2026-11.md`](docs/REVIEW-REMEDIATION-2026-11.md)).
- **C3** — added `PRIVACY.md` and linked it from `config.ini` and
  `tosapp.json` `help`.
- **C9** — rewrote the Chinese descriptions (they previously contained text from
  another application) and made every description match the real features:
  one-time/single-view links, 1 hour/1 day/1 week expiry, optional password.
- **C10** — added this repository layout and the Privacy section; removed stale
  build-probe metadata from `tosapp.json`.
- **App Center installation** — moved the host port from 8142 to 8180 to avoid
  the port already used by another catalog app (`nzbhydradocker`), switched the
  dependency to the short compose form, and made the app health check exercise
  the real memcached connection.

## License / branding

Yopass is distributed by its upstream authors under **Apache-2.0**. The container image is the official image from Docker Hub (`jhaals/yopass:16.2.0`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

- The application runs inside a container; data lives in the application data directory (`/Volume1/DockerAppData/yopassdocker/`) as described above.
- Pending secrets are held in memcached memory and are lost on a container restart; uploaded files persist under `data/files` until they expire.
- The web UI is served over plain HTTP on the LAN. Put it behind HTTPS if it is
  exposed beyond a trusted network.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
