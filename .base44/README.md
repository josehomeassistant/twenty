# Base44 development preview

Run from the repository root:

```sh
docker compose -f docker-compose.base44.yml up -d --build
docker compose -f docker-compose.base44.yml ps -a
docker compose -f docker-compose.base44.yml logs -f setup
```

The initial startup takes roughly ten minutes. The one-shot `setup` service installs dependencies with the repository-pinned Yarn and `--immutable`, builds the required workspace libraries and server, then runs the existing idempotent schema setup and instance migrations. It does **not** reset or seed the database. Application services wait for it to exit successfully.

All application code comes from the bind-mounted checkout, using a plain Node 24 image. Vite serves live frontend source on host port **3000**, and proxies the existing API paths to the private Nest service. Nest watches server source; the worker watches the resulting compiled output. PostgreSQL and Redis are private Compose services with persistent volumes. Local application storage stays in the checkout's ignored `.local-storage` directory.

## Environment and secrets

- `APP_SECRET` is generated securely for development and loaded from the platform-managed `/run/base44/app.env`. It is never recorded in this repository or overridden in Compose. Review/replace it for a production deployment.
- No third-party credentials are needed for core CRM startup. AI providers, OAuth, billing and real email delivery need their own credentials before use. Email and campaign delivery use the existing logging drivers; they do not send real messages.
- Compose supplies local database credentials, internal Redis/API addresses, and public frontend/server URLs derived from `BASE44_PUBLIC_HOST_SUFFIX`.
- Vite receives the platform's `__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS` so the preview proxy's rotating hostnames are accepted without a permissive wildcard.
- `BASE44_PREVIEW_MODE` is passed through, never hardcoded. No shared application configuration changes or flag-dependent overrides were needed: enabled or unset, application code retains its original behavior. All sandbox-specific values are isolated in this Compose file.
- Polling is enabled for bind-mounted source watchers. Nx's background daemon, isolated plugin processes and task cache are disabled for predictable container startup and fresh shared builds.

## Verify and work

```sh
curl -f http://localhost:3000/
curl -f http://localhost:3000/client-config
docker compose -f docker-compose.base44.yml logs --tail=40 api web worker
```

`setup` must show `Exited (0)`; database, Redis, API, frontend and worker should show healthy. The worker probe checks Redis for an active blocking queue consumer, not merely a running PID. The frontend probe checks both HTML and the proxied client configuration.

The anonymous preview opens `/welcome` with **Continue with Email**. Create your account/workspace there; no seeded password is assumed. Initial verification covered this welcome screen with no browser errors or failed requests; signed-in CRM flows require a user session and were not automatically verified.

Frontend and server edits reload automatically. Shared library changes require rebuilding their package; use the repository's commands inside a container. For dependency updates, stop `web`, `worker` and `api`, then run Compose up again so the single setup installer finishes before applications resume. Never remove the data volumes to refresh dependencies.

This is a development environment, not a production deployment.
