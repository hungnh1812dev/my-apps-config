# Configuration: variables, ConfigMaps, Secrets and image tags

No project values are committed to this repository. The manifests in `apps/` use `${APP_*}`
placeholders, and Flux fills them in from values that are either on the cluster or in the sync
files.

## Where each value comes from

| Kind | Where it lives | Read by | Holds |
|---|---|---|---|
| Shared ConfigMap `shared-config` | the environment's namespace `<ns>`, applied by hand | Flux (`postBuild.substituteFrom`, first) | Values all three apps use: name, namespace, env, domain, TLS issuer |
| Per-app ConfigMap `<svc>-config` | `<ns>`, applied by hand | Flux (`postBuild.substituteFrom`, second) | Values that differ per app: service name, image repository, port |
| `postBuild.substitute` in `cluster/me/<env>/<svc>-sync-overlay.yaml` | git, the environment's branch | Flux | `APP_IMAGE_TAG` only |
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
        name: shared-config
      - kind: ConfigMap
        name: cms-api-config
    substitute:                          # from cluster/me/<env>/cms-api-sync-overlay.yaml
      APP_IMAGE_TAG: "81-47e63ce-amd64"
```

Both ConfigMaps are required (`optional` defaults to false). If either is missing, the Kustomization
fails with `ConfigMap ... not found` instead of rendering empty names.

Flux reads `substituteFrom` from the Kustomization's own namespace, which is the namespace the
environment's Flux instance was bootstrapped into ([flux-system.md](flux-system.md)). The names are
fixed: the namespace keeps environments and projects apart.

### Rules

- **Quote every value** in the ConfigMaps (`APP_PORT: "3000"`). ConfigMap data must be strings.
- **`APP_NAMESPACE`-`APP_ENV` must equal the Flux namespace.** Flux, its ConfigMaps and all three
  apps share one namespace per environment. Set both only in the shared ConfigMap, never per app.
- **Use a different `APP_DOMAIN` per environment** on a shared cluster (e.g. `staging.example.com`),
  or the staging and prod Ingresses claim the same hosts.
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
| Flux instance, GitRepository, root Kustomization | `<APP_NAMESPACE>-<APP_ENV>` (set by `flux bootstrap --namespace`) |
| Shared Flux ConfigMap | `shared-config` (fixed), in `<ns>` |
| Per-app Flux ConfigMap | `<svc>-config` (fixed), in `<ns>` |
| Flux Kustomization | `<svc>-sync` (fixed), in `<ns>` |


The Secret templates don't go through Flux, so their `metadata.name` (`<svc>-<env>-secrets`) and
`namespace` have to be written out by hand to match this scheme.

Changing `APP_NAME` or `APP_SERVICE_NAME` on a live cluster renames the app resources: Flux prunes
the old ones and creates new ones, but the hand-made Secrets have to be recreated under the new names
first. `APP_NAMESPACE` and `APP_ENV` are tied to the Flux namespace: changing them means a new Flux
instance ([DEPLOYMENT.md](../DEPLOYMENT.md), steps 15 and 5).

## ConfigMap vs Secret

| | ConfigMap | Secret |
|---|---|---|
| Namespace | `<ns>` | `<ns>` |
| Contains | Non-secret project info (`APP_*`), shared + per-app | App settings and credentials (DB password, JWT keys, `AUTH_SECRET`, ...) |
| Consumer | Flux, at render time | The pods, at start |
| Services | all three | cms-api, frontend (cms-admin has its API URL built into the image) |
| After a change | re-apply, then `flux -n <ns> reconcile kustomization <svc>-sync` (for the shared ConfigMap: all three) | re-apply, then `kubectl -n <ns> rollout restart deploy/<svc>-<env>` |

A ConfigMap change only takes effect after Flux re-renders, on the next interval or a manual
reconcile. A Secret change only takes effect when the pods restart, because `envFrom` is read once
at container start.

## Image-tag contract with the app repository

The app repository's pipeline builds and pushes an image, then commits the new tag here, on the
environment's branch. Flux applies it on its next poll. The contract:

- **File:** `cluster/me/<env>/<svc>-sync-overlay.yaml`, one per service (`cms-api`, `cms-admin`,
  `frontend`) and environment.
- **Branch:** `staging` for `cluster/me/staging/`, `main` for `cluster/me/prod/`. Each environment's
  tags live in their own files, so merging `staging` into `main` never changes a prod tag.
- **Line:** `      APP_IMAGE_TAG: "<tag>"` under `spec.postBuild.substitute`, with the tag in double quotes.
- **Exactly one line per file starts with `APP_IMAGE_TAG:`** (after indentation). The pipeline
  rewrites it by pattern, so a second such line would be rewritten too. Mentions inside
  comments don't match, because they don't start the line.
- **cms-api:** the pipeline must push both `<repo>:<tag>` and `<repo>:<tag>-init` before
  committing, or the init container fails with `ImagePullBackOff`.
- **Never reuse a tag.** Tags like `81-47e63ce-amd64` include the commit SHA, so each one is unique.

An example rewrite (GNU sed, as on Linux CI runners; on macOS use `sed -i ''`):

```bash
sed -i -E 's/^( *APP_IMAGE_TAG: ).*/\1"81-47e63ce-amd64"/' cluster/me/prod/cms-api-sync-overlay.yaml
```

The overlay is applied by the environment's root Kustomization, which then updates the app's
Kustomization. That change triggers a reconcile of the app, which rolls out the new image. To roll
back, revert the tag commit (see [DEPLOYMENT.md](../DEPLOYMENT.md)).
