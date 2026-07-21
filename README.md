# Umbrel Community App Store — TRMNL & OmniRoute

This repository packages three TRMNL Build-Your-Own-Server (BYOS) applications and the OmniRoute AI Gateway as an [Umbrel Community App Store](https://github.com/getumbrel/umbrel-community-app-store):

| App            | Upstream                                                                  | Stack                              | Port  |
|----------------|---------------------------------------------------------------------------|------------------------------------|-------|
| **OmniRoute**  | [diegosouzapw/OmniRoute](https://github.com/diegosouzapw/OmniRoute) — TS  | OmniRoute + Redis                  | 20128 |
| **Terminus**   | [usetrmnl/terminus](https://github.com/usetrmnl/terminus) — Ruby/Hanami   | Terminus + PostgreSQL + Valkey      | 2300  |
| **LaraPaper**  | [usetrmnl/larapaper](https://github.com/usetrmnl/larapaper) — PHP/Laravel  | LaraPaper (bundled SQLite)         | 8080  |
| **Inker**      | [usetrmnl/inker](https://github.com/usetrmnl/inker) — TypeScript/NestJS   | Inker (bundled SQLite + Chromium)  | 8090  |

The TRMNL apps let you manage [TRMNL](https://trmnl.com) e-ink display devices on your own network with full ownership of your data, while OmniRoute provides a powerful self-hosted AI gateway.

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
| `web`      | `ghcr.io/usetrmnl/terminus:latest` | Hanami/Puma web server on port `2300` (runs DB migrations on first boot via `APP_SETUP: true`). |
| `worker`   | `ghcr.io/usetrmnl/terminus:latest` | Sidekiq background jobs (firmware sync, screen rendering, etc.). |
| `database` | `postgres:18.4-alpine`             | PostgreSQL 18.4 database.                         |
| `keyvalue` | `valkey/valkey:9-alpine`           | Valkey 9 (Redis-compatible) cache + job queue.    |

Umbrel's `app_proxy` fronts the `web` service at port `2300`.

**First-run setup:** open the Terminus web UI from the umbrelOS app tile and click **Register**. The **first registered user becomes the verified admin**; subsequent registrations require manual verification. No fixed default credentials.

**Configuration:** sensitive values (`APP_SECRET`, `DATABASE_PASSWORD`, `KEYVALUE_PASSWORD`, `API_URI`) are overridable via a `.env` file. See [`trmnl-terminus/.env.example`](trmnl-terminus/.env.example). The inline defaults are insecure placeholders so the stack boots standalone — **override `APP_SECRET`, `DATABASE_PASSWORD`, and `KEYVALUE_PASSWORD` for any real deployment.** Note `APP_SECRET` must be ≥ 64 chars (Hanami enforces `min_size: 64`). Changing `DATABASE_PASSWORD`/`KEYVALUE_PASSWORD` after first boot requires re-seeding the corresponding volume — see upstream [Docker docs](https://github.com/usetrmnl/terminus/blob/main/doc/docker.adoc#troubleshooting).

---

## LaraPaper

The `trmnl-larapaper` app is a **single container** (no separate DB/cache):

| Service | Image                                | Purpose                                 |
|---------|--------------------------------------|-----------------------------------------|
| `app`   | `ghcr.io/usetrmnl/larapaper:latest`  | Laravel + nginx + php-fpm on port `8080`, bundled SQLite, runs queue worker + migrations automatically via `AUTORUN_ENABLED`. |

Two volumes are persisted:
- `database` → `/var/www/html/database/storage` (SQLite database file)
- `storage`  → `/var/www/html/storage/app/public/images/generated` (generated screen images)

**First-run setup:** open the LaraPaper web UI and **register** the first account (registration enabled by default). The first registered user is the admin. Default credentials `admin@example.com` / `admin@example.com` only apply to dev mode — in production you register your own. Set `REGISTRATION_ENABLED=0` after setup to lock down signups, then point your TRMNL device at this server (firmware ≥ 1.4.6 → "Custom Server" during WiFi setup).

**Configuration:** LAP_KEY (`base64:` + 32 random bytes) should be unique per install — generate with `echo "base64:$(openssl rand -base64 32)"`. See [`trmnl-larapaper/.env.example`](trmnl-larapaper/.env.example).

---

## Inker

The `trmnl-inker` app is an **all-in-one container** (nginx + NestJS backend + bundled SQLite + headless Chromium for BMP rendering, no external DB/cache):

| Service | Image                  | Purpose                                      |
|---------|------------------------|----------------------------------------------|
| `app`   | `wojooo/inker:latest` | E-ink device manager + screen designer (container port `80`, Umbrel port `8090`), bundled SQLite, auto-migrates schema on startup. |

One volume is persisted:
- `uploads` → `/app/uploads` (SQLite database `inker.db` + user uploads: screens, firmware, widgets, captures, drawings)

**First-run setup:** open the Inker web UI and log in with the default PIN **`1111`**. Change it via the `ADMIN_PIN` env var for any real deployment. Note: `ADMIN_PIN` must be quoted in YAML (leading zeros are stripped otherwise) — the compose file already quotes it.

**Configuration:** see [`trmnl-inker/.env.example`](trmnl-inker/.env.example) for `ADMIN_PIN`, `TZ`, `DEFAULT_TIMEZONE` overrides.

---

## OmniRoute

The `omniroute` app provides the OmniRoute AI Gateway:

| Service | Image                                | Purpose                                 |
|---------|--------------------------------------|-----------------------------------------|
| `web`   | `diegosouzapw/omniroute:latest-web`  | OmniRoute dashboard (port `20128`) and API (port `20129`). Includes Chromium for web-cookie providers. |
| `redis` | `docker.io/library/redis:7-alpine`   | Redis backend for rate limiting.        |

One volume is persisted:
- `omniroute-data` → `/app/data` (OmniRoute data and configuration)

**First-run setup:** open the OmniRoute web UI from the umbrelOS app tile to access the dashboard and configure your AI providers. The dashboard runs on port `20128` via the proxy, and the API is directly exposed on port `20129`.

**Configuration:** port bindings and origins are configurable via a `.env` file. See [`trmnl-omniroute/.env.example`](trmnl-omniroute/.env.example) for defaults.

---

## Source

- Terminus upstream: https://github.com/usetrmnl/terminus — MIT
- LaraPaper upstream: https://github.com/usetrmnl/larapaper — MIT
- Inker upstream: https://github.com/usetrmnl/inker — AGPL-3.0
- OmniRoute upstream: https://github.com/diegosouzapw/OmniRoute — MIT

This packaging repo is unaffiliated with the upstream maintainers; it only wraps their published container images for Umbrel.