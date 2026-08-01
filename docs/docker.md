# Running with Docker

The repo ships a multi-stage `Dockerfile` (Next.js standalone output,
runs as a non-root user) and a `docker-compose.yml` with a single
`app` service. Supabase is external — point the app at your hosted
(or self-hosted) Supabase project via env vars; no database container
is included.

## Quick start

1. Copy the env template and fill it in:

   ```bash
   cp .env.local.example .env.local
   ```

2. Build and start (the `--env-file` flag is required — Compose only
   reads `.env` by default for `${VAR}` substitution, and this project
   keeps its config in `.env.local`):

   ```bash
   docker compose --env-file .env.local up --build -d
   ```

3. The app is served on [http://localhost:3000](http://localhost:3000)
   (publish it elsewhere with `HOST_PORT=8080` in `.env.local`).

> Use `HOST_PORT`, not `PORT`, to move the published port. `PORT` is
> what the server listens on _inside_ the container, and `env_file`
> would inject it there — leaving the app on a port the mapping and
> the healthcheck don't target. Compose pins it to 3000 for that
> reason.

## Build-time vs runtime variables

- `NEXT_PUBLIC_*` variables are **inlined into the client bundle at
  build time**. They are passed as Docker build args by
  `docker-compose.yml`. If you change any of them, rebuild:
  `docker compose --env-file .env.local up --build -d`.
- Everything else (`SUPABASE_SERVICE_ROLE_KEY`, `ENCRYPTION_KEY`,
  `META_APP_SECRET`, …) is read at **runtime** from `.env.local` via
  `env_file` and is never baked into the image — safe to change with
  just a container restart.

## Plain Docker (no Compose)

```bash
docker build \
  --build-arg NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co \
  --build-arg NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key \
  -t wacrm .

docker run -d --env-file .env.local -e PORT=3000 -p 3000:3000 wacrm
```

## Deploy on Easypanel

Easypanel builds directly from the `Dockerfile` in this repo — it does
**not** read `docker-compose.yml`, so build args and runtime secrets
both need to be entered in its UI instead of `.env.local`.

1. **Create App → From a Git repository**, pointing at your fork.
   Easypanel detects the `Dockerfile` automatically (build method
   "Dockerfile").
2. **Build args** (Easypanel's *Build* tab → *Build Arguments*) — add
   these `NEXT_PUBLIC_*` values, since they're inlined at build time
   and unreachable at runtime:
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `NEXT_PUBLIC_SITE_URL` (your Easypanel domain, e.g. `https://crm.yourdomain.com`)
   - `NEXT_PUBLIC_APP_LOCALE` (optional, defaults to `en`)
3. **Environment variables** (Easypanel's *Environment* tab) — add
   the runtime-only secrets instead: `SUPABASE_SERVICE_ROLE_KEY`,
   `ENCRYPTION_KEY`, `META_APP_SECRET`, and any optional vars from
   `.env.local.example` you use. These are read at container start,
   never baked into the image, so changing them just needs a restart
   (no rebuild).
4. **Domain / port** — set the container port to `3000` when adding
   your domain; the image listens there (`EXPOSE 3000` /
   `HOSTNAME=0.0.0.0`). Easypanel terminates TLS, so no extra config
   is needed for HTTPS.
5. Deploy. The image ships a Docker `HEALTHCHECK` that Easypanel picks
   up automatically to report container health.
6. As with any deployment, database migrations under `supabase/` and
   the cron endpoints described below are your responsibility —
   Easypanel doesn't run either automatically.

## Notes

- Database migrations under `supabase/` are **not** run by the
  container — apply them with the Supabase CLI as described in the
  README.
- Nothing inside the container is scheduled. If you use automation
  Wait steps or flows, point an external scheduler at
  `GET /api/automations/cron` and `GET /api/flows/cron` on this
  deployment, sending the shared secret in the `x-cron-secret` header
  (`AUTOMATION_CRON_SECRET`, see `.env.local.example`). Both return
  503 until that variable is set.
