# cms-api

The CMS backend: a Bun server with Prisma on Postgres, serving REST under `/api/v1` and GraphQL.
The cms-admin SPA calls it from the browser, and the frontend calls it from its server.

- **Base:** `apps/cms-api/`
- **Flux Kustomization:** `project-me-cms-api-sync-prod` (`cluster/me/cms-api-sync.yaml`)
- **ConfigMaps:** `project-me-prod-shared-config` (template `templates/shared-configmap.example.yaml`)
  and `project-me-cms-api-prod-config` (template `templates/cms-api-configmap.example.yaml`)
- **Secret:** `<APP_SERVICE_NAME>-<APP_ENV>-secrets` (template `templates/cms-api-secret.example.yaml`)

Names below use `<name>` for `<APP_SERVICE_NAME>-<APP_ENV>` and `<ns>` for
`<APP_NAMESPACE>-<APP_ENV>` (see [configuration.md](configuration.md)).

## Resources

| Kind | Name | File |
|---|---|---|
| Deployment | `<name>` | `apps/cms-api/deployment.yaml` |
| Service (ClusterIP) | `<name>` | `apps/cms-api/service.yaml` |
| Ingress | `<name>` | `apps/cms-api/ingress.yaml` |
| Middleware (Traefik) | `<name>-https-redirect` | `apps/cms-api/middleware.yaml` |

All four are in `apps/cms-api/kustomization.yaml`, the same layout as cms-admin and frontend.

## Networking

| | Value |
|---|---|
| Host | `api.<APP_DOMAIN>`, all paths |
| TLS | `<name>-tls`, issued by cert-manager from `APP_TLS_CLUSTER_ISSUER` |
| HTTP | permanently redirected to HTTPS by the Middleware |
| Port | `APP_PORT` for the container, the Service (`http`) and `PORT` in the app |
| In-cluster URL | `http://<name>.<ns>.svc.cluster.local:<APP_PORT>` |

The container command is `sh -c "PORT=${APP_PORT} exec bun dist/src/main"`. Setting `PORT` there
overrides any `PORT` in the Secret, so the app and the Service always use the same port. The
command must stay in sync with the `CMD` in the app repository's Dockerfile.

## Startup and probes

1. **Init container `init`** (`<APP_IMAGE_REPO>:<APP_IMAGE_TAG>-init`) runs `prisma migrate deploy`
   against the database in the Secret. If migrations fail, the app container never starts.
2. **Container `app`** (`<APP_IMAGE_REPO>:<APP_IMAGE_TAG>`) starts the server.

| Probe | Check | Initial delay | Period |
|---|---|---|---|
| Readiness | `GET /health` on `APP_PORT` | 5s | 10s |
| Liveness | `GET /health` on `APP_PORT` | 15s | 20s |

`/health` is served outside the `/api/v1` prefix.

## Resources (CPU / memory)

| Container | Requests | Limits |
|---|---|---|
| `init` | 50m / 128Mi | 500m / 512Mi |
| `app` | 100m / 128Mi | 500m / 512Mi |

## Configuration

**ConfigMaps (for Flux):** the shared one holds `APP_NAME`, `APP_NAMESPACE`, `APP_ENV`, `APP_DOMAIN`
and `APP_TLS_CLUSTER_ISSUER`. cms-api's own holds `APP_SERVICE_NAME`, `APP_IMAGE_REPO` and `APP_PORT`.
`APP_IMAGE_TAG` is in the sync file.

**Secret (for the app, via `envFrom` on both containers):**

| Group | Keys | Required |
|---|---|---|
| JWT | `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET` | yes |
| Cookie | `COOKIE_SECURE`, `COOKIE_SAMESITE` | |
| CORS | `CORS_ORIGINS` (comma-separated exact origins, e.g. `https://admin.<domain>`) | yes |
| Proxy | `TRUST_PROXY` (`1` = the Traefik hop) | |
| Database | `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD` | yes |
| Storage | `STORAGE_PROVIDER` + `CLOUDINARY_*` or `AWS_*` | |
| Email | `EMAIL_PROVIDER`, `EMAIL_FROM` + provider keys (`SMTP_*`, `GMAIL_*`, `RESEND_API_KEY`, `BREVO_API_KEY`), `FRONTEND_URL` | |
| Redis | `REDIS_ENABLED`, `REDIS_URL` (required if enabled) | |
| Limits | `RATE_LIMIT_FPS`, `RATE_LIMIT_BURST`, `MEDIA_MAX_UPLOAD_BYTES`, `CONTENT_TYPES_DIR` | |

Leave optional keys out rather than setting them to `""`. An empty value counts as set, and some
keys fail validation so the app won't boot. Don't set `PORT`. The full list is in
`apps/cms-api/.env.example` in the app repository.

## Security

- `automountServiceAccountToken: false`: the app never calls the Kubernetes API.
- Pod `seccompProfile: RuntimeDefault`.
- Both containers: `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`.
- No `runAsNonRoot` yet, because the image's `USER` is set in the app repository
  ([AUDIT.md](../AUDIT.md), finding 4).

## Dependencies

- **Postgres** outside the cluster, reachable from the pod at `DB_HOST:DB_PORT`.
- **Traefik** (k3s bundled) and **cert-manager** with the ClusterIssuer named in the shared ConfigMap.
- **The namespace and Secret** must exist before the first reconcile.
- **Images:** the app repository must push both `<tag>` and `<tag>-init`.

## Known limits

- One replica, so each rollout has a short gap while the new pod runs migrations and becomes ready.
  Two replicas would both run `prisma migrate deploy` at start ([AUDIT.md](../AUDIT.md), finding 15).
- Migrations run on every pod start, not once per release. Prisma skips migrations that were already
  applied, so a restart is safe, but a slow database delays each start.
- Rolling back the image doesn't roll back the database. A release whose migration isn't
  backward-compatible can't be undone by reverting the tag.
