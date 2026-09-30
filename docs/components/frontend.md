# frontend

The public website: a Next.js standalone server with Auth.js. It renders pages from cms-api
content, which it fetches over GraphQL from its server.

- **Base:** `apps/frontend/`
- **Flux Kustomization:** `frontend-sync` in the environment's namespace (`cluster-base/frontend-sync.yaml`,
  image tag in `cluster/me/<env>/frontend-sync-overlay.yaml`)
- **ConfigMaps:** `shared-config` (template `templates/shared-configmap.example.yaml`)
  and `frontend-config` (template `templates/frontend-configmap.example.yaml`), both in `<ns>`
- **Secret:** `<APP_SERVICE_NAME>-<APP_ENV>-secrets` (template `templates/frontend-secret.example.yaml`)

Names below use `<name>` for `<APP_SERVICE_NAME>-<APP_ENV>` (see
[configuration.md](configuration.md)).

## Resources

| Kind | Name | File |
|---|---|---|
| Deployment | `<name>` | `apps/frontend/deployment.yaml` |
| Service (ClusterIP) | `<name>` | `apps/frontend/service.yaml` |
| Ingress | `<name>` | `apps/frontend/ingress.yaml` |
| Middleware (Traefik) | `<name>-https-redirect` | `apps/frontend/middleware.yaml` |

## Networking

| | Value |
|---|---|
| Host | `<APP_DOMAIN>` (the bare domain), all paths |
| TLS | `<name>-tls`, issued by cert-manager from `APP_TLS_CLUSTER_ISSUER` |
| HTTP | permanently redirected to HTTPS by the Middleware |
| Port | `APP_PORT` (3000) for the container (named `http`) and the Service |

The image sets `PORT=3000` and Next listens there. `APP_PORT` only tells Kubernetes that port, so it
must equal the image's `PORT`. The Ingress host uses `${APP_DOMAIN:=APP_DOMAIN-is-not-set}`. A
bare `${APP_DOMAIN}` that was left empty would drop the host and match every request. The
placeholder isn't a valid hostname, so the apply fails instead.

## Probes

| Probe | Check | Initial delay | Period | Timeout |
|---|---|---|---|---|
| Readiness | `GET /api/health` on `http` | — | 10s | 8s |
| Liveness | TCP connect on `http` | 10s | 20s | 1s (default) |

These are chosen so that problems with cms-api don't take the frontend down:

- `/api/health` always returns 200 and reports cms-api's status in the body, so readiness only
  checks that the Next server answers.
- The route stops waiting for cms-api after 5s. The 8s probe timeout is longer, so a hanging cms-api
  doesn't mark the pod unready.
- Liveness is a TCP check only, so a slow cms-api never gets the frontend restarted.

## Resources (CPU / memory)

Requests 100m / 192Mi, limits 500m / 512Mi.

## Configuration

**ConfigMaps (for Flux):** the shared one holds `APP_NAME`, `APP_NAMESPACE`, `APP_ENV`, `APP_DOMAIN`
and `APP_TLS_CLUSTER_ISSUER`. The frontend's own holds `APP_SERVICE_NAME`, `APP_IMAGE_REPO` and
`APP_PORT`. `APP_IMAGE_TAG` is in the sync file.

**Set in the Deployment:**

| Env | Value | Why |
|---|---|---|
| `AUTH_TRUST_HOST` | `"true"` | Auth.js v5 behind a proxy (Traefik) has to trust the forwarded host |
| `HOSTNAME` | `"0.0.0.0"` | Next binds to `$HOSTNAME`, and the runtime sets it to the pod name |

**Secret (via `envFrom`):**

| Key | Required | Notes |
|---|---|---|
| `AUTH_SECRET` | yes | Auth.js session secret; the server throws without it |
| `CMS_API_URL` | yes | cms-api base for login/refresh/logout |
| `GRAPHQL_URL` | yes | `<cms-api-url>/graphql` |
| `REVALIDATE_SECRET` | yes | Shared with cms-api, which calls `/api/revalidate` to purge cached pages |
| `NEXT_ENV` | | `production` turns off dev-only request logging |
| `GRAPHQL_TOKEN`, `STRAPI_API_TOKEN` | | Optional bearer tokens for GraphQL |

Don't set `PORT` or `AUTH_TRUST_HOST` in the Secret. For the cms-api URLs, prefer the in-cluster
Service over the public host, so requests don't go out through Traefik and back:
`http://cms-api-<APP_ENV>.<APP_NAMESPACE>-<APP_ENV>.svc.cluster.local:<cms-api APP_PORT>`.

The public GraphQL URL used by the health check is built into the image (`FRONTEND_GRAPHQL_URL` in
the app repository), not set here.

## Security

- `automountServiceAccountToken: false`.
- Pod `seccompProfile: RuntimeDefault`.
- Container: `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`.
- No `runAsNonRoot` yet, because the image's `USER` is set in the app repository
  ([AUDIT.md](../AUDIT.md), finding 4).

## Dependencies

- **cms-api**, for content and auth at request time. The frontend still starts and stays Ready
  without it, but pages that need content fail. There's no Flux `dependsOn`
  ([AUDIT.md](../AUDIT.md), finding 12).
- **Traefik** and **cert-manager** with the ClusterIssuer named in the shared ConfigMap.
- **The namespace and Secret** must exist before the first reconcile.

## Known limits

- The sync file's `APP_IMAGE_TAG` is `dev` until the app repository commits a real tag
  ([AUDIT.md](../AUDIT.md), finding 9).
- One replica, so a rollout or a node drain has a short gap.
- Only the bare domain is served. `www.<domain>` needs its own DNS record and Ingress host.
