# Change report: docker-compose setup for CISO Assistant

Date: 2026-11-05. Based on the upstream
[`docker-compose.yml`](../Resources/repos/ciso-assistant-community/docker-compose.yml)
and the helper script
[`docker-compose.sh`](../Resources/repos/ciso-assistant-community/docker-compose.sh).

## Goal

Run CISO Assistant from a single `docker-compose.yml` (suitable as a Dockhand stack):

- no helper bash/PowerShell scripts,
- Docker named volumes instead of the `./db` bind mount,
- fix the sqlite3 database creation failure.

## Root cause of the sqlite3 / huey failure

1. The backend image does not contain `/code/db` (the backend `.dockerignore` excludes `db`).
2. When a fresh volume (or bind mount) is mounted at `/code/db`, Docker creates it **owned by root**.
3. The containers run as user `1001:1001` and cannot write there, so SQLite cannot create
   `ciso-assistant.sqlite3` or `huey.db`.
4. `huey` (the task worker) fails to start; because the backend/other services depend on a
   working stack, the whole thing shuts down.

Upstream works around this in `docker-compose.sh`, which runs `mkdir ./db` and `chown -R 1001:1001 ./db`
on the host. That cannot work with named volumes (and not on Docker Desktop for Windows).

## What was done and why

| Change | Why |
| --- | --- |
| Added one-shot `init` service (busybox, runs as root) | Replaces the `chown` from `docker-compose.sh`. Sets ownership of the `ciso_db` volume to 1001:1001, creates `attachments/`. `backend` uses `depends_on: service_completed_successfully`, so it only starts afterwards. |
| `init` also generates `/db/django_secret_key` | `backend/startup.sh` normally does this with `openssl`, but it needs a writable `db/`. Doing it in `init` removes the race and keeps a persistent key. |
| `./db:/code/db` replaced by named volume `ciso_db` (backend + huey) | Requested; works in Dockhand and survives `docker compose down`. |
| `./db/caddy:/data/caddy` replaced by named volume `caddy_data:/data` | Same reason. Persists Caddy's internal CA so the certificate stays stable. Mounting `/data` (not `/data/caddy`) is Caddy's standard layout. |
| `huey` entrypoint exports `DJANGO_SECRET_KEY` from the volume | Huey bypasses `startup.sh`, so without this it would use a different random key than the backend (`settings.py` falls back to `get_random_secret_key()`). |
| `DJANGO_SUPERUSER_EMAIL` / `DJANGO_SUPERUSER_PASSWORD` on backend | `startup.sh` runs `createsuperuser --noinput` when the email is set. Replaces the interactive `docker compose exec backend python manage.py createsuperuser` step of `docker-compose.sh`. Defaults are `admin@example.com` / `ChangeMe-123!`. |
| Backend healthcheck uses `localhost:8000` instead of `backend:8000` | Does not depend on service-name DNS inside the container. |
| `CISO_ASSISTANT_URL`, `CA_MCP_ALLOWED_HOSTS` configurable via `${VAR:-default}` | Lets Dockhand / a `.env` file change the URL without editing the file. |
| YAML anchors (`x-backend-env`, `x-hardening`) | Removes duplicated env/hardening blocks. Security hardening (read-only FS, dropped caps, no-new-privileges, tmpfs `/tmp`) is kept from upstream. |
| `mcp` service kept as opt-in (`profiles: [mcp]`) and `/mcp*` route kept in Caddy | Same as upstream. Only started with `--profile mcp`. |
| Not changed: `frontend`, `qdrant`, image tags | Unchanged from upstream. |

## Verification

Tested locally with Docker 29.9.2 on Windows from a clean state (`docker compose -p cisotest ...`):

- `docker compose config` valid.
- `init` exited 0; `/code/db` owned by `app` (1001) containing `ciso-assistant.sqlite3`, `huey.db`,
  `django_secret_key`, `attachments/`.
- `backend` healthy, `huey` running and executing scheduled tasks, `frontend`, `caddy`, `qdrant` up.
- `https://localhost:8443/` returned `302` (to login), `https://localhost:8443/api/health/` returned `{"status":"ok"}`.
- With `--profile mcp`: `mcp` container healthy, `https://localhost:8443/mcp` reachable through Caddy
  (an empty JSON-RPC POST returns a validation error from the MCP server, as expected).
- Test stack and its volumes were removed afterwards.

Not tested: login in a browser with the auto-created superuser, and running inside Dockhand.

## Notes / caveats

- Default superuser password is public in this file: set your own via `.env` or Dockhand variables
  before first start. `createsuperuser` only runs when the user is created; changing the variables later does not
  change an existing password (and the startup command may log an "already exists" error, which is harmless).
- `pull_policy: always` pulls `:latest` images on every `up`. Pin tags for reproducible deployments.
- `DJANGO_DEBUG=True` on the backend is kept from upstream; set it to `False` for anything beyond local development.
- `docker compose down -v` deletes all data (database, secret key, attachments, Caddy CA).
