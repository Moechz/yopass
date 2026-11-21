# Privacy Policy — Yopass for TerraMaster TOS

Last updated: 2026-11-21

- **App ID:** `yopassdocker`
- **Package:** Yopass for TerraMaster TOS 7
- **Publisher:** Moechz
- **Package source:** <https://github.com/Moechz/yopass>
- **Upstream project:** <https://github.com/jhaals/yopass>

## 1. Scope and data controller

This policy describes the Yopass application package for TerraMaster TOS 7
("the app"). Yopass is a **self-hosted** application: it runs entirely on the
NAS where it is installed, using the NAS owner's Docker engine. There is no
hosted Yopass service operated by the publisher.

The person or organisation that installs and administers the NAS is the data
controller for the secrets processed by this app. The publisher does not receive
secrets, encryption keys, account data, logs or telemetry from the app, and has
no access to the NAS.

## 2. What the app processes

When you create a secret, the app processes:

- the **encrypted secret** — text, or an uploaded file;
- **metadata** needed to deliver it: the random secret identifier, the chosen
  lifetime, whether it is one-time, whether a password protects it, and the
  creation time (and file size, for uploads);
- **ordinary container logs** produced by the Docker runtime on your NAS.

The app does not ask for a name, e-mail address or account. It does not create
user profiles, does not contain analytics, advertising, crash reporting or
telemetry, and does not contact the publisher.

## 3. End-to-end encryption and key handling

Encryption and decryption happen **in the browser** with OpenPGP
(AES-GCM/AEAD, as implemented by the upstream Yopass web client):

- The browser generates the data-encryption key. The optional custom password
  is used locally to protect that key.
- The key (and the password, when one is used) is placed in the **URL fragment**
  — the part after `#` in the generated link. Browsers do not send the fragment
  to the server, so the server never receives the key or password.
- The server receives and stores only the **ciphertext** and the metadata listed
  above. It has no way to decrypt the secret.
- The link itself is the secret: anyone who obtains a complete link can open the
  secret, subject to its one-time/password/expiry settings. Share it only over a
  channel you trust.

## 4. Storage and retention

- **Text secrets** are kept in the app's private **memcached** container. That
  container has no published host port and is reachable only by the app
  container on the internal Docker network.
- **Uploaded files** are kept as ciphertext on disk under
  `/Volume1/DockerAppData/yopassdocker/data` (container path `/data/files`).
- **Lifetime:** the app offers 1 hour, 1 day or 1 week. A secret is deleted when
  its lifetime expires. A **one-time** secret is deleted as soon as it is read
  successfully.
- The file store runs a cleanup pass about once a minute and removes expired
  files. Pending secrets are also lost when the memcached or app container is
  recreated or restarted; this is normal for an in-memory store.

## 5. Logs

The app writes standard container logs (stdout/stderr) that are kept by the NAS
Docker runtime and can be read by the NAS administrator with
`docker logs yopassdocker`. These logs contain operational metadata (for
example startup messages, request paths and errors). They do **not** contain
secret plaintext, decryption keys or passwords. The package does not ship logs
to the publisher or to any third party. Log retention is controlled by the NAS
administrator and the Docker logging configuration.

## 6. Accounts, cookies and telemetry

The package runs Yopass with its default local settings: **no account system, no
OIDC login, no audit logging, no webhooks and no read receipts**. It sets no
tracking cookies and loads no third-party scripts, fonts, analytics or CDNs.
The browser may keep ordinary local preferences (such as theme or language
choice) in local storage; these never leave the device.

## 7. Sharing and third parties

The publisher does not share data with third parties. The app itself sends
nothing to the publisher. Two external services are involved only in operating
the package:

- **Docker Hub** (`docker.io`) — the container images are pulled from Docker Hub
  when the app is installed or updated, subject to Docker's own privacy policy.
- **GitHub** — the package source and this policy are hosted on GitHub, subject
  to GitHub's privacy policy.

No other third-party service is contacted by the application.

## 8. Security measures

- Client-side, end-to-end encryption; keys stay in the URL fragment.
- The main container and the memcached container run as **uid 1000:1000**, never
  as root.
- Fixed image tag (no `:latest`), images used unmodified from Docker Hub.
- No `privileged`, no `network_mode: host`, no Docker socket, no added
  capabilities. Only the web port is published; memcached is internal-only.
- Persistent data stays below `/Volume1/DockerAppData/yopassdocker/`.
- Plain HTTP on the LAN by default. If you expose the app beyond your trusted
  network, put it behind HTTPS (reverse proxy or VPN) — otherwise the link,
  including its fragment, can be observed in transit.

## 9. Your rights and deletion

Because the app has no accounts, there is no server-side identity to export or
correct. You can control the data directly:

- **Delete one secret:** use the delete action on the result page, or simply let
  it expire; a one-time secret disappears after its first read.
- **Delete everything:** stop and remove the app, then remove
  `/Volume1/DockerAppData/yopassdocker/data` (choose "delete data" when
  uninstalling, or delete the folder manually). Restarting the containers clears
  the in-memory memcached store as well.
- **Logs:** clear or rotate the container logs through the NAS Docker tooling.

For questions about this policy or the package, open an issue at
<https://github.com/Moechz/yopass/issues>. For the upstream application, see
<https://github.com/jhaals/yopass>.

## 10. Children

The app is a general-purpose secret-sharing tool for NAS administrators and is
not directed at children.

## 11. Changes to this policy

This policy is versioned with the package. Material changes will be published in
this file and noted in `CHANGELOG.md`; the "Last updated" date at the top is
changed at the same time.
