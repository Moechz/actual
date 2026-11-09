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

## License / branding

Actual Budget is distributed by its upstream authors under **MIT**. The container image is the official image from Docker Hub (`actualbudget/actual-server:26.11.0`) and is used unmodified; no upstream source code is redistributed here. This packaging is an independent community submission by Moechz and is not affiliated with the upstream project or TerraMaster.

## Limitations

- The web UI is served over plain HTTP on the published port; use a TLS reverse proxy if you expose it outside your LAN.
- The server has no account system by default - set the server password on first use, otherwise anyone who can reach the port can open your budgets.
- The data folder is not visible in TOS File Manager; back it up over SMB/SSH or add the path to a TOS backup task.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
