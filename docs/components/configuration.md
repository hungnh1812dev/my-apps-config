# Configuration: variables, ConfigMaps, Secrets and image tags

No project values are committed to this repository. The manifests in `apps/` use `${APP_*}`
placeholders, and Flux fills them in from values that are either on the cluster or in the sync
files.

## Where each value comes from

| Kind | Where it lives | Read by | Holds |
|---|---|---|---|
| Shared ConfigMap `project-me-prod-shared-config` | `flux-system` namespace, applied by hand | Flux (`postBuild.substituteFrom`, first) | Values all three apps use: name, namespace, env, domain, TLS issuer |
| Per-app ConfigMap `project-me-<svc>-prod-config` | `flux-system` namespace, applied by hand | Flux (`postBuild.substituteFrom`, second) | Values that differ per app: service name, image repository, port |
| `postBuild.substitute` in `cluster/me/<svc>-sync.yaml` | git | Flux | `APP_IMAGE_TAG` only |
| Secret `<svc>-<env>-secrets` | app namespace, applied by hand | the app's pods (`envFrom`) | Runtime config and secrets (cms-api and frontend only) |

The ConfigMaps are only for Flux: substitution happens before anything reaches the cluster, so
the pods never see them. The Secrets are only for the apps: Flux never reads them.

Templates for all six are in `templates/*.example.yaml`. Copy one to the same name without
`.example`, fill it in, and apply it with `kubectl apply --server-side -f`. The filled-in copies
are gitignored.

## Variables

| Variable | Source | cms-api | cms-admin | frontend | Used for |
|---|---|---|---|---|---|
| `APP_NAME` | shared ConfigMap | ✓ | ✓ | ✓ | The `app.kubernetes.io/name` label |
| `APP_NAMESPACE` | shared ConfigMap | ✓ | ✓ | ✓ | Namespace base (the apps run in `<APP_NAMESPACE>-<APP_ENV>`) |
| `APP_ENV` | shared ConfigMap | ✓ | ✓ | ✓ | Environment suffix on names and the namespace, e.g. `prod` |
| `APP_DOMAIN` | shared ConfigMap | ✓ | ✓ | ✓ | Bare domain. Hosts: `api.<domain>`, `admin.<domain>`, `<domain>` |
| `APP_TLS_CLUSTER_ISSUER` | shared ConfigMap | ✓ | ✓ | ✓ | cert-manager ClusterIssuer on each Ingress |
| `APP_SERVICE_NAME` | per-app ConfigMap | ✓ | ✓ | ✓ | First part of every resource name. Must differ per service |
| `APP_IMAGE_REPO` | per-app ConfigMap | ✓ | ✓ | ✓ | Image repository without a tag |
| `APP_PORT` | per-app ConfigMap | ✓ | — | ✓ | Container and Service port (cms-api also gets it as `PORT`) |
| `APP_IMAGE_TAG` | sync file | ✓ | ✓ | ✓ | Image tag (cms-api's init image uses `<tag>-init`) |

cms-admin listens on 80, fixed by its nginx image, so it has no `APP_PORT`. The frontend image sets
`PORT=3000` itself, so its `APP_PORT` must be `"3000"`: it tells Kubernetes the port, it doesn't
change it.

Each sync file lists the shared ConfigMap first and the per-app one second. When a key is in both,
the later one wins, so an app can override a shared value by adding the key to its own ConfigMap:

```yaml
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: project-me-prod-shared-config
      - kind: ConfigMap
        name: project-me-cms-api-prod-config
    substitute:
      APP_IMAGE_TAG: "81-47e63ce-amd64"
```

Both ConfigMaps are required (`optional` defaults to false). If either is missing, the Kustomization
fails with `ConfigMap ... not found` instead of rendering empty names.

### Rules

- **Quote every value** in the ConfigMaps (`APP_PORT: "3000"`). ConfigMap data must be strings.
- **Put shared values only in the shared ConfigMap.** All three apps run in one namespace, which is
  created by hand ([DEPLOYMENT.md](../DEPLOYMENT.md), step 6), so `APP_NAMESPACE` and `APP_ENV` must
  not be overridden per app.
- **Use a different `APP_SERVICE_NAME`** per service, e.g. `cms-api`, `cms-admin`, `frontend`.
  Two services with the same value would render resources with the same name. The frontend Secret
  template's in-cluster cms-api URL assumes cms-api uses `cms-api`.
- **Set `APP_DOMAIN` before the first apply.** A missing value fails the apply instead of
  exposing a catch-all Ingress. cms-api and cms-admin get the invalid host `api.` / `admin.`, and
  the frontend gets `APP_DOMAIN-is-not-set`.
- **Don't put `APP_IMAGE_TAG` in a ConfigMap.** `postBuild.substitute` takes precedence over
  `substituteFrom`, so it would be ignored.
- A variable missing from both sources becomes an empty string. It doesn't fail the render.

## Naming scheme

Every name is derived from the variables, so the manifests contain no project values:

| Thing | Name |
|---|---|
| Namespace (all three apps) | `<APP_NAMESPACE>-<APP_ENV>` |
| Deployment, Service, Ingress | `<APP_SERVICE_NAME>-<APP_ENV>`, e.g. `cms-api-prod` |
| Label `app.kubernetes.io/name` (selectors) | `<APP_NAME>-<APP_SERVICE_NAME>-<APP_ENV>` |
| Runtime Secret (cms-api, frontend) | `<APP_SERVICE_NAME>-<APP_ENV>-secrets` |
| TLS Secret (created by cert-manager) | `<APP_SERVICE_NAME>-<APP_ENV>-tls` |
| Traefik Middleware | `<APP_SERVICE_NAME>-<APP_ENV>-https-redirect` |
| Ingress middleware annotation | `<APP_NAMESPACE>-<APP_ENV>-<APP_SERVICE_NAME>-<APP_ENV>-https-redirect@kubernetescrd` |
| Shared Flux ConfigMap (in `flux-system`) | `project-me-prod-shared-config` (fixed in the sync files) |
| Per-app Flux ConfigMap (in `flux-system`) | `project-me-<svc>-prod-config` (fixed in the sync files) |
| Flux Kustomization (in `flux-system`) | `project-me-<svc>-sync-prod` (fixed) |

The ConfigMap templates use placeholder names (`<app-name>-<app-env>-shared-config`,
`<app-name>-<app-service-name>-<app-env>-config`). Fill them in so they become the fixed names above.

The Secret templates don't go through Flux, so their `metadata.name` (`<svc>-<env>-secrets`) and
`namespace` have to be written out by hand to match this scheme.

Changing `APP_NAME`, `APP_SERVICE_NAME`, `APP_NAMESPACE` or `APP_ENV` on a live cluster renames
everything. Flux prunes the old resources and creates new ones, but the hand-made namespace and
Secrets have to be recreated under the new names first.

## ConfigMap vs Secret

| | ConfigMap | Secret |
|---|---|---|
| Namespace | `flux-system` | the app namespace |
| Contains | Non-secret project info (`APP_*`), shared + per-app | App settings and credentials (DB password, JWT keys, `AUTH_SECRET`, ...) |
| Consumer | Flux, at render time | The pods, at start |
| Services | all three | cms-api, frontend (cms-admin has its API URL built into the image) |
| After a change | re-apply, then `flux reconcile kustomization project-me-<svc>-sync-prod` (for the shared ConfigMap: all three) | re-apply, then `kubectl -n <ns> rollout restart deploy/<svc>-<env>` |

A ConfigMap change only takes effect after Flux re-renders, on the next interval or a manual
reconcile. A Secret change only takes effect when the pods restart, because `envFrom` is read once
at container start.

## Image-tag contract with the app repository

The app repository's pipeline builds and pushes an image, then commits the new tag to `main` here.
Flux applies it on its next poll. The contract:

- **File:** `cluster/me/<svc>-sync.yaml`, one per service (`cms-api`, `cms-admin`, `frontend`).
- **Line:** `      APP_IMAGE_TAG: "<tag>"` under `spec.postBuild.substitute`, with the tag in double quotes.
- **Exactly one line per file starts with `APP_IMAGE_TAG:`** (after indentation). The pipeline
  rewrites it by pattern, so a second such line would be rewritten too. Mentions inside
  comments don't match, because they don't start the line.
- **cms-api:** the pipeline must push both `<repo>:<tag>` and `<repo>:<tag>-init` before
  committing, or the init container fails with `ImagePullBackOff`.
- **Never reuse a tag.** Tags like `81-47e63ce-amd64` include the commit SHA, so each one is unique.

An example rewrite (GNU sed, as on Linux CI runners; on macOS use `sed -i ''`):

```bash
sed -i -E 's/^( *APP_IMAGE_TAG: ).*/\1"81-47e63ce-amd64"/' cluster/me/cms-api-sync.yaml
```

The sync file is applied by the root `flux-system` Kustomization, which then updates the app's
Kustomization. That change triggers a reconcile of the app, which rolls out the new image. To roll
back, revert the tag commit (see [DEPLOYMENT.md](../DEPLOYMENT.md)).
