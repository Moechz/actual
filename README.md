# actual-for-tos

> TerraMaster TOS 7 App Center — **Docker application** package for
> [Actual Budget](https://github.com/actualbudget/actual).

## What this is

A four-file TOS Docker application archive (`actualbudgetdocker.tar.gz`) that installs
Actual Budget on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `actualbudgetdocker`)
- `actualbudgetdocker.lang` — 23-language superset (TOS requires 14)
- `actualbudgetdocker.svg` — application icon (274 bytes, 2 SVG elements)
- `docker-compose.yml` — one service, image `actualbudget/actual-server:26.11.0`

## How it works

| | |
|---|---|
| Container image | `actualbudget/actual-server:26.11.0` (Docker Hub, used unmodified) |
| Open | `http://<NAS-IP>:8033/` (container port 5006) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/data` → `/Volume1/DockerAppData/actualbudgetdocker/data` |
| Logs | `docker logs actualbudgetdocker` (stdout/stderr) |

`/data` (mounted from `/Volume1/DockerAppData/actualbudgetdocker/data`) holds the server files and every budget file you create: `server-files/` keeps the sync state and `user-files/` the budgets and uploaded statements.

## Privacy

This application runs entirely on your NAS. **The developer does not collect, receive or
have access to any data processed by the application.** There is no telemetry, no analytics
and no account.

| Topic | Detail |
|---|---|
| Data processed | The application's configuration, database, caches and logs, plus any files you create with it. |
| Where it is stored | The application data folder (`/Volume1/DockerAppData/actualbudgetdocker/`) on your NAS. Nothing is uploaded to the developer. |
| Retention period | Until **you** delete it. Logs are rotated or size-limited by the application; uninstalling keeps the data unless you tick “delete data”. |
| Your rights | **Access / export**: copy the data folder. **Correct**: change the values in the web UI. **Delete**: uninstall with “delete data”, or remove the folder after stopping the container. |
| Deletion channel | Delete the folder yourself, or email **moechz@163.com** (answered within 30 days). |
| Security measures | Container runs as uid 1000 (never root), no `privileged`, no host network, no `cap_add`, no Docker socket, one published port; image tag pinned and refreshed on upstream fixes. |
| Children | Not directed at children. |

**Full privacy policy:** [PRIVACY.md](PRIVACY.md) —
<https://github.com/Moechz/actual/blob/main/PRIVACY.md> — and the website version at
<https://www.moechz.com/privacy.html>.

## Third-party services (API disclosure)

The developer operates no server in this path. The application contacts only the services
**you configure**, plus (optionally) an update check and the image registry.

| Service | Endpoint | Data sent | Destination / region |
|---|---|---|---|
| Services you configure in the app | the address you enter | only the data that feature needs | normally your own LAN or the provider you chose |
| Update check (if enabled) | the upstream project's release feed | current version string | GitHub, US |
| Image pull | `registry-1.docker.io` or your registry mirror | anonymous pull | Docker Hub, US / your mirror |

See `PRIVACY.md` §6 for the full table. Disable any integration in the app's settings if you
do not want it.

## Licence and attributions

| Component | Licence | Notes |
|---|---|---|
| Actual Budget | MIT | upstream application, used unmodified inside the image |
| Container image `actualbudget/actual-server:26.11.0` | see the image's own metadata | pulled from Docker Hub at install time |
| This packaging | GPL-3.0 (see `LICENSE`) | compose file, metadata, translations, icon |

“Actual Budget”, “TerraMaster” and “TOS” are trademarks of their respective owners. This is an
independent community package by Moechz and is not affiliated with the upstream project or
TerraMaster. Full details: [NOTICE.md](NOTICE.md).

## Design highlights

- **Non-root.** `user: "1000:1000"`; verified on real hardware (the container reports uid 1000).
- **Fixed image tag, never `:latest`**, Docker Hub only, no `privileged`, no `network_mode: host`, and no Docker socket, no `cap_add`.
- **Health check** probing the service on container port 5006.
- **Deterministic build**: GNU tar with uid/gid 0 and zeroed timestamps, gzip without a timestamp, plus the review-standards self-check above.
- **No database server.** Actual stores budgets in SQLite files under `/data`; the container starts as uid 1000 and needs nothing else.
- **One volume, one port.** Only `/data` is persistent and only container port 5006 is published.
- **Interpreter healthcheck.** The image ships neither curl nor wget, so the healthcheck asks the bundled Node runtime to fetch the start page.

## Runtime file manifest

Everything the application writes on the NAS lives inside its own data
directory; the container-internal `/tmp` is discarded with the container. The
table lists what is created and how it behaves. Nothing else is written.

| Path | Purpose | Created when | Growth bound | Lifecycle |
|---|---|---|---|---|
| `/Volume1/DockerAppData/actualbudgetdocker/data` | Actual Budget application data (everything the container writes outside /tmp) | on first start / on demand | bounded by your own data | persistent; kept on uninstall unless you ask to delete it |

## Build

```bash
scripts/build.sh                  # -> out/actualbudgetdocker.tar.gz + .sha256
scripts/build.sh 26.11.0-2   # packaging iteration bump
scripts/build.sh 26.11.0-1 aarch64
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
./TOSAppSelfTestingTool -c out/actualbudgetdocker.tar.gz   # official spec check
bash test/install-and-verify.sh                  # on-device E2E (needs root)
```

> Docker applications cannot be side-loaded through the App Center's manual
> install page (it accepts `.deb` only); the official self-testing tool's `-i`
> mode or `docker compose` are the only ways to install one for testing.

## Store submission checklist

1. Public GitHub repo (code + README only).
2. Release **tag `v26.11.0-1`**, assets `actualbudgetdocker.tar.gz` +
   `actualbudgetdocker.tar.gz.sha256` (Docker asset naming: `<app_id>.tar.gz`, **no
   platform suffix**).
3. Developer platform → Add Application: ID `actualbudgetdocker`, package type Docker,
   repository URL, architecture `x86_64`.
4. Version Management → Add Version `26.11.0-1` → automated validation → review.


## Limitations

- The web UI is served over plain HTTP on the published port; use a TLS reverse proxy if you expose it outside your LAN.
- The server has no account system by default - set the server password on first use, otherwise anyone who can reach the port can open your budgets.
- The data folder is not visible in TOS File Manager; back it up over SMB/SSH or add the path to a TOS backup task.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.

## Support

- Issues and questions: <https://github.com/Moechz/actual/issues>
- Email: moechz@163.com
- Privacy: [PRIVACY.md](PRIVACY.md) · Licence: [NOTICE.md](NOTICE.md)
