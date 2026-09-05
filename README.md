# Srihari's — an Umbrel Community App Store

This repository packages three TRMNL Build-Your-Own-Server (BYOS) applications, two self-hosted AI gateways, and the Cronicle task scheduler as an [Umbrel Community App Store](https://github.com/getumbrel/umbrel-community-app-store):

| App             | Upstream                                                                  | Stack                              | Port  |
|-----------------|---------------------------------------------------------------------------|------------------------------------|-------|
| **OmniRoute**   | [diegosouzapw/OmniRoute](https://github.com/diegosouzapw/OmniRoute) — TS  | OmniRoute + Redis                  | 20128 |
| **VansRouter**  | [Vanszs/VansRouter](https://github.com/Vanszs/VansRouter) — JS/Next.js    | VansRouter (bundled SQLite) + Headroom | 20130 |
| **Terminus**    | [usetrmnl/terminus](https://github.com/usetrmnl/terminus) — Ruby/Hanami   | Terminus + PostgreSQL + Valkey      | 2300  |
| **LaraPaper**   | [usetrmnl/larapaper](https://github.com/usetrmnl/larapaper) — PHP/Laravel  | LaraPaper (bundled SQLite)         | 8080  |
| **Inker**       | [usetrmnl/inker](https://github.com/usetrmnl/inker) — TypeScript/React    | Inker (bundled SQLite + Chromium)  | 8090  |
| **Cronicle**    | [jhuckaby/Cronicle](https://github.com/jhuckaby/Cronicle) — Node.js       | Cronicle (bundled storage)         | 3012  |

The TRMNL apps let you manage [TRMNL](https://trmnl.com) e-ink display devices on your own network with full ownership of your data, OmniRoute and VansRouter put one endpoint in front of many AI providers, and Cronicle schedules and runs jobs across one or more machines.

## Add this app store to Umbrel

In the umbrelOS UI:

1. Open **Settings → Community App Stores → Add**.
2. Enter this repository's GitHub URL.
3. Once the store appears, browse it and install any of the available apps.

---

## Terminus

The `trmnl-terminus` app runs these Docker Compose services:

| Service    | Image                              | Purpose                                          |
|------------|------------------------------------|--------------------------------------------------|
| `web`      | `ghcr.io/usetrmnl/terminus:0.72.0` | Hanami/Puma web server on port `2300` (runs DB migrations on first boot via `APP_SETUP: true`). |
| `worker`   | `ghcr.io/usetrmnl/terminus:0.72.0` | Sidekiq background jobs (firmware sync, screen rendering, etc.). |
| `database` | `postgres:18.6-alpine`             | PostgreSQL 18.6 database.                         |
| `keyvalue` | `valkey/valkey:9-alpine`           | Valkey 9 (Redis-compatible) cache + job queue.    |

Umbrel's `app_proxy` fronts the `web` service at port `2300`.

**First-run setup:** open the Terminus web UI from the umbrelOS app tile and click **Register**. The **first registered user becomes the verified admin**; subsequent registrations require manual verification. No fixed default credentials.

**Configuration:** sensitive values (`APP_SECRET`, `DATABASE_PASSWORD`, `KEYVALUE_PASSWORD`, `API_URI`) are overridable via a `.env` file. See [`trmnl-terminus/.env.example`](trmnl-terminus/.env.example). The inline defaults are insecure placeholders so the stack boots standalone — **override `APP_SECRET`, `DATABASE_PASSWORD`, and `KEYVALUE_PASSWORD` for any real deployment.** Note `APP_SECRET` must be ≥ 64 chars (Hanami enforces `min_size: 64`). Changing `DATABASE_PASSWORD`/`KEYVALUE_PASSWORD` after first boot requires re-seeding the corresponding volume — see upstream [Docker docs](https://github.com/usetrmnl/terminus/blob/main/doc/docker.adoc#troubleshooting).

---

## LaraPaper

The `trmnl-larapaper` app is a **single container** (no separate DB/cache):

| Service | Image                                | Purpose                                 |
|---------|--------------------------------------|-----------------------------------------|
| `app`   | `ghcr.io/usetrmnl/larapaper:0.41.0`  | Laravel + nginx + php-fpm on port `8080`, bundled SQLite, runs queue worker + migrations automatically via `AUTORUN_ENABLED`. |

Three volumes are persisted:
- `database`      → `/var/www/html/database/storage` (SQLite database file)
- `image_assets`  → `/var/www/html/storage/app/public/images` (shipped default assets, including the scheduled-sleep screen)
- `storage`       → `/var/www/html/storage/app/public/images/generated` (generated screen images)

To replace the scheduled-sleep screen, copy your own file into the running container rather than bind-mounting over that directory — a bind mount would hide the shipped defaults behind an empty host directory:

```
docker cp sleep.png trmnl-larapaper_app_1:/var/www/html/storage/app/public/images/sleep.png
```

**First-run setup:** open the LaraPaper web UI and **register** the first account (registration enabled by default). The first registered user is the admin. Default credentials `admin@example.com` / `admin@example.com` only apply to dev mode — in production you register your own. Set `REGISTRATION_ENABLED=0` after setup to lock down signups, then point your TRMNL device at this server (firmware ≥ 1.4.6 → "Custom Server" during WiFi setup).

**Configuration:** `APP_KEY` (`base64:` + 32 random bytes) should be unique per install — generate with `echo "base64:$(openssl rand -base64 32)"`. See [`trmnl-larapaper/.env.example`](trmnl-larapaper/.env.example).

---

## Inker

The `trmnl-inker` app is an **all-in-one container** (nginx + TypeScript/React backend + bundled SQLite + headless Chromium for BMP rendering, no external DB/cache):

| Service | Image                  | Purpose                                      |
|---------|------------------------|----------------------------------------------|
| `app`   | `wojooo/inker:0.6.0`   | E-ink device manager + screen designer (container port `80`, Umbrel port `8090`), bundled SQLite, auto-migrates schema on startup. Supports TRMNL OG and TRMNL X. |

One volume is persisted:
- `uploads` → `/app/uploads` (SQLite database `inker.db` + user uploads: screens, firmware, widgets, captures, drawings)

**First-run setup:** open the Inker web UI and log in with the default PIN **`1111`**. Change it via the `ADMIN_PIN` env var for any real deployment. Note: `ADMIN_PIN` must be quoted in YAML (leading zeros are stripped otherwise) — the compose file already quotes it.

**Configuration:** see [`trmnl-inker/.env.example`](trmnl-inker/.env.example) for `ADMIN_PIN`, `TZ`, `DEFAULT_TIMEZONE` overrides.

---

## OmniRoute

The `omniroute` app provides the OmniRoute AI Gateway:

| Service | Image                                  | Purpose                                 |
|---------|----------------------------------------|-----------------------------------------|
| `web`   | `diegosouzapw/omniroute:3.8.50-web`    | OmniRoute dashboard (port `20128`) and API (port `20129`). Includes Chromium for web-cookie providers. |
| `redis` | `docker.io/library/redis:8.6.5-alpine` | Redis backend for rate limiting.        |

One volume is persisted:
- `omniroute-data` → `/app/data` (OmniRoute data and configuration)

**First-run setup:** open the OmniRoute web UI from the umbrelOS app tile to access the dashboard and configure your AI providers. The dashboard runs on port `20128` via the proxy, and the API is directly exposed on port `20129`.

**Configuration:** port bindings and origins are configurable via a `.env` file. See [`trmnl-omniroute/.env.example`](trmnl-omniroute/.env.example) for defaults. Note that `LIVE_WS_ALLOWED_ORIGINS` defaults to `*` here so the WebSocket works from any hostname you reach Umbrel by; set it to your actual origins if you want that tightened.

Upstream's `web` build profile also ships `chatgpt-web-codex-browser` and `codex-app-server` sidecars. Neither is packaged here — they exist for running from source, and the Chromium sidecar would add a 2 GB shm container to every install. The ChatGPT Web (Codex) provider therefore will not work in this packaging.

---

## VansRouter

The `trmnl-vansrouter` app is a second AI gateway — a fork of [9Router](https://github.com/decolua/9router) that also pulls in OmniRoute's routing logic. It runs alongside OmniRoute rather than replacing it; the two are independent installs with their own data.

| Service    | Image                                     | Purpose                                                                 |
|------------|-------------------------------------------|-------------------------------------------------------------------------|
| `app`      | `ghcr.io/vanszs/vansrouter:0.91.21`       | Next.js dashboard **and** the OpenAI/Anthropic/Gemini-compatible API, both on container port `20128`. Bundled SQLite. |
| `headroom` | `ghcr.io/chopratejas/headroom:0.6.0-slim` | Optional compression sidecar for Token Saver → Headroom.                 |

Data is bind-mounted rather than kept in a named volume, so umbrelOS backs it up:
- `${APP_DATA_DIR}/data/vansrouter` → `/app/data` (`db/data.sqlite`, the generated `jwt-secret` and `auth/api-key-secret`, certs, logs)

**First-run setup:** open the app and sign in with the password umbrelOS shows on the app page, then change it under **Settings**. There is no username — login is password-only. The tile opens at `/masuk` (the app's login route) because VansRouter's `/` returns a JSON welcome payload rather than redirecting to the login page.

**Using it from your editor:** create an API key under **Endpoint & Key**, then point your tools at `http://umbrel.local:20130/v1` with that key. The key is mandatory here even though a laptop install can go without one: VansRouter treats anything that is not a loopback caller as remote, and every request through Umbrel's proxy is remote by that definition.

### Packaging notes

Six packaging decisions worth knowing about, most of them forced by how VansRouter behaves behind a reverse proxy:

- **Port `20130`, not `20128`.** VansRouter and OmniRoute both listen on `20128` internally, and umbrelOS host ports have to be unique. Only the `app_proxy` port moved; the container still listens on `20128`. `20130` is unused in the official app store.
- **Umbrel auth off** (`PROXY_AUTH_ADD: "false"`), as with every other app here. The dashboard and the API share a single port, and with Umbrel auth on, a request without an umbrelOS session cookie is answered with a 302 to the login page. Coding agents can't carry that cookie, so they follow the redirect and get HTML instead of a completion — the API is unusable. `PROXY_AUTH_WHITELIST` can exempt individual path prefixes, but it isn't worth relying on when the whole point of the app is that arbitrary clients reach `/v1`. What still guards the app is VansRouter's own bcrypt password login with per-IP lockout, plus its API-key check on `/v1` — it treats every non-loopback caller as remote, and everything through the proxy is remote, so a key is mandatory.
- **`INITIAL_PASSWORD` must be set.** VansRouter refuses a login that still uses its built-in `123456` when the request looks remote, and asks you to change it "from the local machine" — which, behind `app_proxy`, is nowhere. Left alone the app would be permanently unreachable. The compose file wires `INITIAL_PASSWORD` to Umbrel's per-install `${APP_PASSWORD}` and the manifest sets `deterministicPassword: true`, so umbrelOS generates and displays a unique password instead.
- **Images are pinned by digest** as well as tag, following the official app-store rule. Upstream publishes bare version tags on GHCR (`0.91.21`, not `v0.91.21`) and nothing at all on Docker Hub, despite what its `DOCKER.md` says.
- **Bind mount instead of a named volume**, so the SQLite database is inside `app-data` and covered by umbrelOS backups. Keeping the secrets in that directory rather than deriving them from the device seed is deliberate: it means a backup restored onto a different Umbrel keeps working with the API keys you already issued.
- **No extra capabilities.** The image bundles `tailscaled` and would want `CAP_NET_ADMIN` for TUN mode. It isn't granted, and nothing is lost: the tunnel endpoints are unreachable anyway (see below).

**What doesn't work in this packaging:** the built-in Cloudflare and Tailscale tunnels, the MCP endpoints, the Cursor/Kiro CLI auto-import, in-app password reset, and starting Headroom from the dashboard. VansRouter restricts those routes to loopback callers because they spawn host processes or read host secrets, and requests through `app_proxy` never qualify. Use umbrelOS's own remote access to reach the dashboard from outside your network. The in-app updater under Settings is best left alone too — it would patch a container filesystem that is discarded on the next restart; update through the umbrelOS App Store instead.

The `headroom` sidecar is genuinely optional and idles until you switch it on. It is included because upstream's own compose file ships it and because the in-app installer is one of the loopback-only routes, so a sidecar is the only way to reach that feature from a browser. The headline RTK, Caveman and Ponytail savings work without it. Nothing else in the compose file references it, so deleting the service is enough if you'd rather not run it.

**Configuration:** see [`trmnl-vansrouter/.env.example`](trmnl-vansrouter/.env.example). Anything in upstream's `.env.example` can be set there; `DATA_DIR`, `PORT`, `HOSTNAME`, `NODE_ENV` and `INITIAL_PASSWORD` are fixed by the package.

---

## Cronicle

The `trmnl-cronicle` app is a **single container**:

| Service | Image                        | Purpose                                                     |
|---------|------------------------------|-------------------------------------------------------------|
| `app`   | `soulteary/cronicle:0.9.80`  | Multi-server task scheduler and runner with a web UI on port `3012`. |

Three volumes are persisted:
- `data`    → `/opt/cronicle/data` (schedule, users, job history)
- `logs`    → `/opt/cronicle/logs` (job and server logs)
- `plugins` → `/opt/cronicle/plugins` (custom plugins)

**First-run setup:** open the Cronicle web UI and log in with **`admin` / `admin`**, then change the password immediately via **My Account**.

**Configuration:** `TZ` sets the container timezone (defaults to `Asia/Kolkata`). See [`trmnl-cronicle/.env.example`](trmnl-cronicle/.env.example).

The container's entrypoint clears a stale `cronicled.pid` before starting, which is what allows the app to survive an unclean shutdown. Note that the packaged image is pinned to Cronicle 0.9.80 because `soulteary/cronicle` has not been rebuilt since June 2025; upstream Cronicle is further ahead.

---

## Store assets

Gallery images follow the convention used by the official Umbrel App Store
(`getumbrel/umbrel-apps-gallery`): **exactly three images per app, 16:10, JPEG,
RGB**, at 1440x900 — the 1x size the official store uses for apps like Home
Assistant and Nextcloud. Each card is a brand-coloured backdrop, one short
headline sentence, and a real screenshot inside a macOS-style browser window
that bleeds off the bottom edge.

They live in `<app>/gallery/{1,2,3}.jpg` and are referenced by raw URL, because
community app stores supply `gallery:` and `icon:` as URLs rather than the bare
filenames official packages use. 1440x900 was chosen over the newer 2880x1800
because the upstream screenshots behind these cards top out around 1500–1920px
wide; rendering at 2x would only upscale them.

Icons are square and fully opaque, so nothing shows the wallpaper through the
tile. Where a project's own logo is transparent or circular, it is composited
onto a rounded tile and committed here; otherwise the manifest links the
project's own icon at a pinned tag.

Terminus publishes no screenshots in its own repo, so its three cards are built
from real UI captures its maintainers attached to
[#332](https://github.com/usetrmnl/terminus/issues/332),
[#324](https://github.com/usetrmnl/terminus/issues/324) and
[#288](https://github.com/usetrmnl/terminus/issues/288). One had a transient
"Serverless transforms are not supported yet." error toast from the bug it was
filed against; that band was cut out so the nav sits directly above the page
content. The permanent notices on those pages are left as they are.

VansRouter publishes exactly one screenshot (`images/9router.png`), so its three
cards are the full Providers page plus two tighter crops of it at different zoom
levels. Crop edges land in the gutters between provider cards so none is sliced
in half.

---

## Source

- Terminus upstream: https://github.com/usetrmnl/terminus — MIT
- LaraPaper upstream: https://github.com/usetrmnl/larapaper — MIT
- Inker upstream: https://github.com/usetrmnl/inker — AGPL-3.0
- OmniRoute upstream: https://github.com/diegosouzapw/OmniRoute — MIT
- VansRouter upstream: https://github.com/Vanszs/VansRouter — MIT (a fork of [decolua/9router](https://github.com/decolua/9router); the optional sidecar is [chopratejas/headroom](https://github.com/chopratejas/headroom))
- Cronicle upstream: https://github.com/jhuckaby/Cronicle — MIT (packaged image: [soulteary/docker-cronicle](https://github.com/soulteary/docker-cronicle))

This packaging repo is unaffiliated with the upstream maintainers; it only wraps their published container images for Umbrel.