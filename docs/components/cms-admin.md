# cms-admin

The CMS admin UI: a static single-page app served by nginx. It runs entirely in the browser and
calls cms-api at `api.<domain>`.

- **Base:** `apps/cms-admin/`
- **Flux Kustomization:** `project-me-cms-admin-sync-prod` (`cluster/me/cms-admin-sync.yaml`)
- **ConfigMaps:** `project-me-prod-shared-config` (template `templates/shared-configmap.example.yaml`)
  and `project-me-cms-admin-prod-config` (template `templates/cms-admin-configmap.example.yaml`)
- **Secret:** none

Names below use `<name>` for `<APP_SERVICE_NAME>-<APP_ENV>` (see
[configuration.md](configuration.md)).

## Resources

| Kind | Name | File |
|---|---|---|
| Deployment | `<name>` | `apps/cms-admin/deployment.yaml` |
| Service (ClusterIP) | `<name>` | `apps/cms-admin/service.yaml` |
| Ingress | `<name>` | `apps/cms-admin/ingress.yaml` |
| Middleware (Traefik) | `<name>-https-redirect` | `apps/cms-admin/middleware.yaml` |

## Networking

| | Value |
|---|---|
| Host | `admin.<APP_DOMAIN>`, all paths |
| TLS | `<name>-tls`, issued by cert-manager from `APP_TLS_CLUSTER_ISSUER` |
| HTTP | permanently redirected to HTTPS by the Middleware |
| Port | container `80` (named `http`), Service `80` → `http` |

Port 80 comes from the image's nginx config, so it's fixed in the manifests instead of being a
variable.

## Probes

| Probe | Check | Initial delay | Period |
|---|---|---|---|
| Readiness | `GET /healthz` on `http` | — | 10s |
| Liveness | `GET /healthz` on `http` | 5s | 20s |

nginx answers `/healthz` itself, with no file or upstream involved, so the probes don't depend on
cms-api.

## Resources (CPU / memory)

Requests 10m / 32Mi, limits 200m / 128Mi.

## Configuration

**ConfigMaps (for Flux):** the shared one holds `APP_NAME`, `APP_NAMESPACE`, `APP_ENV`, `APP_DOMAIN`
and `APP_TLS_CLUSTER_ISSUER`. cms-admin's own holds `APP_SERVICE_NAME` and `APP_IMAGE_REPO` (no
`APP_PORT`: nginx's port 80 is fixed). `APP_IMAGE_TAG` is in the sync file.

**Runtime:** nothing. The cms-api URL is built into the image (`VITE_API_URL`, from the
`CMS_ADMIN_API_URL` variable in the app repository). To point cms-admin at a different API,
rebuild the image. Changing the ConfigMap won't do it.

cms-api must list `https://admin.<APP_DOMAIN>` in its `CORS_ORIGINS`, or the browser blocks
every API call.

## Security

- `automountServiceAccountToken: false`.
- Pod `seccompProfile: RuntimeDefault`.
- Container: `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, then adds back
  `NET_BIND_SERVICE`, `CHOWN`, `SETUID`, `SETGID`. Stock nginx starts as root to bind port 80 and
  then switches to its worker user, and it needs those four capabilities to do that.
- No `runAsNonRoot`, because stock nginx runs as root ([AUDIT.md](../AUDIT.md), finding 4).

**Untested:** the capability set hasn't been run against the real image. If the pod crash-loops
with a `chown` or `setgid` permission error, remove the `capabilities` block from
`apps/cms-admin/deployment.yaml`. To fully harden it, switch to `nginxinc/nginx-unprivileged`
(port 8080) as described in the audit.

## Dependencies

- **cms-api** at `api.<APP_DOMAIN>`, for the browser, not for the pod. The pod starts and is Ready
  without it.
- **Traefik** and **cert-manager** with the ClusterIssuer named in the shared ConfigMap.
- **The namespace** must exist before the first reconcile.

## Known limits

- The sync file's `APP_IMAGE_TAG` is `dev` until the app repository commits a real tag. If the
  registry has no `dev` tag, the pod stays in `ImagePullBackOff` ([AUDIT.md](../AUDIT.md), finding 9).
- One replica, so a rollout or a node drain has a short gap.
