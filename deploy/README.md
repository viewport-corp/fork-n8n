# n8n — Viewport deploy overlay

Deploy overlay for **n8n** (workflow automation) on the Viewport infrastructure.
This is the `viewport/deploy` branch of the `viewport-corp/fork-n8n` fork; it adds
**only** a deploy overlay under `deploy/` — upstream n8n source is untouched.

- Sub-issue: viewport-ops **#414** (Part of **#404**)
- Department project: **Automation and Workflows**
- Upstream (product): https://github.com/n8n-io/n8n
- Compose source-of-truth: https://github.com/n8n-io/n8n-hosting/tree/main/docker-compose/withPostgres
- Process: LOCKED #363 fork → clone → upstream → overlay → PR

## What this overlay is

The official, maintained self-host path for n8n is the `withPostgres` Docker Compose
bundle in `n8n-io/n8n-hosting`. That compose is reproduced here verbatim with the
Viewport guardrail edits applied. The n8n product source in this fork is **not** modified.

## Stack (3 services, per current upstream default)

| Service | Image | Tag | Role |
|---------|-------|-----|------|
| `n8n` | `docker.n8n.io/n8nio/n8n` | `2.27.4` | UI / API / scheduler / task broker |
| `postgres` | `postgres` | `16` | persistent datastore |
| `n8n-runner` | `n8nio/runners` | `2.27.4` | external task runner (`N8N_RUNNERS_MODE=external`) |

The runner is part of the current upstream default (external task-runner mode). It is
kept to match upstream. It can be dropped (internal runner mode) if Sam prefers — flagged
for #414 decision, not improvised here.

## Image version

- Pinned tag: **`2.27.4`** — latest stable n8n release. The `2.27.4` tag exists for all
  three images (`n8nio/n8n`, `n8nio/runners`, and the `docker.n8n.io` mirror), so n8n and
  its runner stay version-aligned. Never `:stable`/`:latest` for a stateful service.

## Ports

| Port | Purpose | Exposed |
|------|---------|---------|
| 5678 | n8n HTTP UI + REST API + `/healthz` | internal only (`expose`, no host publish) |
| 5679 | n8n → runner task broker | internal only |
| 5432 | Postgres | internal only (not published) |

The upstream compose publishes `5678:5678` to the host. Per Viewport guardrails (no
`:80`/`:443`, no DNS, no public subdomain this stage) the host publish is **removed** and
replaced with `expose: 5678` — the editor/API is reachable only on the Dokploy network.
No Dokploy Domain, no Traefik labels this stage.

## Persistence (named volumes — required)

- `db_storage` → `/var/lib/postgresql/data` (Postgres data)
- `n8n_storage` → `/home/node/.n8n` (n8n user data **and** the encryption key)
- Bind: `./init-data.sh` → `/docker-entrypoint-initdb.d/init-data.sh` — creates the
  non-root DB user that n8n connects as, on first Postgres init only.

## Configuration / required env (values = Dokploy secrets, names only in git)

Postgres:

| Variable | Purpose |
|----------|---------|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | superuser + database created on first init |
| `POSTGRES_NON_ROOT_USER` / `POSTGRES_NON_ROOT_PASSWORD` | least-privilege user n8n connects as (created by `init-data.sh`) |

n8n:

| Variable | Purpose |
|----------|---------|
| `N8N_VERSION` | image tag — set to `2.27.4` |
| `N8N_ENCRYPTION_KEY` | **CRITICAL.** Encrypts stored credentials. Set before first boot and store as a secret; losing/rotating it permanently locks all saved credentials. |
| `RUNNERS_AUTH_TOKEN` | shared secret between n8n broker and the runner |
| `GENERIC_TIMEZONE` | schedule/cron correctness (also mapped to `TZ`) |

DB connection vars (`DB_TYPE`, `DB_POSTGRESDB_*`) are hardcoded to the `postgres` service
name in the compose (no hardcoded IPs, per guardrails).

Out of scope this stage (set only once a domain exists): `N8N_HOST`, `N8N_PROTOCOL`,
`N8N_PORT`, `WEBHOOK_URL`, `N8N_EDITOR_BASE_URL`.

## Deploy (Dokploy, new engine)

- Dokploy: `http://194.163.153.171:3001/` (engine socket `/var/run/docker-viewport.sock`)
- Project: **Automation and Workflows**
- Path: **Create Service → Compose**, pointed at this fork's `viewport/deploy` branch,
  compose path `deploy/docker-compose.yml`.
- Set all secrets in the Environment tab (names above) — never commit literals.
- Do **not** add a Dokploy Domain (that wires Traefik/`:443`, gated/out of scope).

## Decisions flagged for #414 (not improvised)

1. Pin `N8N_VERSION` — done: `2.27.4` (latest stable).
2. Keep upstream external `n8n-runner` vs collapse to internal runner mode — **kept** (matches upstream); awaiting confirm.
3. `N8N_ENCRYPTION_KEY` — must be generated and stored as a Dokploy secret before first boot.

## Health verification (internal only)

- n8n liveness: `GET http://<n8n-container>:5678/healthz` → `200 {"status":"ok"}`
- Postgres: compose `pg_isready` healthcheck green
- `docker ps` shows all three services up; runner connected to the broker

## Sources (docs-first, verified)

- GitMCP raw fetch of `n8n-io/n8n-hosting/docker-compose/withPostgres/` (`docker-compose.yml`, `.env`, `init-data.sh`)
- Official n8n self-host / configuration env-var docs (docs.n8n.io)
- Official Dokploy Compose-service + named-volume docs
- Live VPS verification: latest stable release tag + image-tag existence on Docker Hub
