# Flux: bootstrap and sync

How Flux gets from this Git repository to the three apps. A cluster can run many Flux instances:
one per project and environment, each bootstrapped into that environment's namespace and watching
only that namespace. For project-me:

| Environment | Namespace (Flux + apps) | Branch | Path |
|---|---|---|---|
| staging | `project-me-staging` | `staging` | `cluster/me/staging` |
| prod | `project-me-prod` | `main` | `cluster/me/prod` |

Another project (another config repository) bootstraps the same way into `<project>-<env>`.

## Resources (per environment, all in `<ns>`)

| Resource | Name | Defined in | What it does |
|---|---|---|---|
| Flux controllers | source-controller, kustomize-controller | `cluster/me/<env>/<ns>/gotk-components.yaml` | The Flux v2.9.5 install. Generated, don't edit |
| Secret | `flux-system` | created by `flux bootstrap` (not in git) | SSH deploy key for GitHub |
| GitRepository | `<ns>` | `cluster/me/<env>/<ns>/gotk-sync.yaml` | Polls the environment's branch every 1m |
| Kustomization | `<ns>` | `cluster/me/<env>/<ns>/gotk-sync.yaml` | The root: applies `./cluster/me/<env>` every 10m |
| Kustomization | `cms-api-sync`, `cms-admin-sync`, `frontend-sync` | `cluster-base/<svc>-sync.yaml` + `cluster/me/<env>/<svc>-sync-overlay.yaml` | Applies `./apps/<svc>` |

`flux bootstrap --namespace=<ns>` names the GitRepository and the root Kustomization `<ns>`, not
`flux-system`, and writes its files to `cluster/me/<env>/<ns>/`, not `flux-system/`.
`cluster/me/<env>/kustomization.yaml` lists `<ns>` as a resource and patches the sync files'
`sourceRef` to match.

## How it fits together

```
GitHub: hungnh1812dev/project-me-config
  │  branch staging                         branch main
  ▼                                         ▼
GitRepository project-me-staging          GitRepository project-me-prod
  ▼                                         ▼
Kustomization project-me-staging          Kustomization project-me-prod
  path ./cluster/me/staging                 path ./cluster/me/prod
  ├── project-me-staging/ (Flux itself)     ├── project-me-prod/
  ├── ../../../cluster-base                 ├── ../../../cluster-base
  │     cms-api-sync ──► apps/cms-api       │     (same three sync files)
  │     cms-admin-sync ─► apps/cms-admin    │
  │     frontend-sync ──► apps/frontend     │
  └── <svc>-sync-overlay.yaml (image tags)  └── <svc>-sync-overlay.yaml
  everything in namespace project-me-staging   everything in namespace project-me-prod
```

`cluster/me/<env>/kustomization.yaml` sets `namespace: <ns>` on everything it renders, so the sync
Kustomizations, the ConfigMaps they read and the apps they deploy all live in the Flux namespace.
Each sync Kustomization renders `apps/<svc>`, substitutes `${APP_*}` from `shared-config` and
`<svc>-config` (see [configuration.md](configuration.md)), and applies the result.

Promotion: changes land on `staging` first, then `staging` is merged into `main`. Image tags live in
per-environment files, so a merge carries the staging tag files along without touching prod's.

## Bootstrap

The full command, with every flag explained, is in [DEPLOYMENT.md](../DEPLOYMENT.md#5-install-the-flux-cli-and-bootstrap).
The flags that make instances independent:

| Flag | Why |
|---|---|
| `--namespace=<ns>` | Flux, its deploy key and its objects live in the environment's namespace |
| `--watch-all-namespaces=false` | The controllers only reconcile Flux objects in `<ns>`. Without it, every instance reconciles every other instance's objects |
| `--network-policy=false` | Flux's default NetworkPolicy denies ingress from other namespaces to every pod in `<ns>`, which would cut Traefik off from the apps |
| `--components=source-controller,kustomize-controller` | Nothing here uses Helm or notifications; two controllers per instance instead of four |
| `--branch`, `--path` | Must match the table above. Any other path rewrites `gotk-sync.yaml` and the apps are no longer applied |

`gotk-components.yaml` and `gotk-sync.yaml` are generated. Never copy them from another environment:
they carry the namespace, branch and flags. `<ns>/kustomization.yaml` is hand-written and kept
by bootstrap.

## Rules for several instances on one cluster

- **One Flux version for all instances.** The CRDs are cluster-wide and every instance applies them.
  Upgrade every instance on the cluster together ([DEPLOYMENT.md](../DEPLOYMENT.md#13-upgrading-flux)).
- **CRDs are never pruned.** `<ns>/kustomization.yaml` marks them
  `kustomize.toolkit.fluxcd.io/prune: disabled`, so removing one instance can't delete the CRDs, which
  would delete every other instance's GitRepositories and Kustomizations.
- **Never run `flux uninstall`** on a shared cluster: it deletes the CRDs. Remove one instance as
  described in [DEPLOYMENT.md](../DEPLOYMENT.md#15-removing-one-instance).
- **Every instance watches only its own namespace.** An instance installed with
  `--watch-all-namespaces=true` (the default) also reconciles every other instance's objects.

## App Kustomization settings

All three sync files share these settings:

| Field | Value | Why |
|---|---|---|
| `interval` | `3m` | Re-applies the rendered manifests, which also reverts manual edits on the cluster |
| `retryInterval` | `1m` | A failed apply is retried sooner than the next interval |
| `prune` | `true` | A resource removed from `apps/<svc>` is deleted from the cluster |
| `wait` | `true` | Ready only once every applied resource is healthy (Deployment rolled out, etc.) |
| `timeout` | `5m` | How long `wait` waits before the reconcile is marked failed |
| `postBuild.substituteFrom` | `shared-config`, then `<svc>-config` | The `${APP_*}` values: shared first, per-app second (per-app wins on a clash) |
| `postBuild.substitute` | `APP_IMAGE_TAG` | The image tag, from `cluster/me/<env>/<svc>-sync-overlay.yaml` |

There's no `dependsOn` between apps. The frontend tolerates cms-api being down, and a `dependsOn`
would block frontend deploys whenever cms-api isn't Ready ([AUDIT.md](../AUDIT.md), finding 12).

## Pruning

Both levels have `prune: true`, so deletions in git propagate:

- Removing a file from `apps/<svc>/kustomization.yaml` deletes that resource.
- Removing a sync file from `cluster-base/kustomization.yaml` deletes that app's Kustomization in
  every environment, and with it every resource the app deployed.

To stop an app from changing without deleting it, suspend it instead:

```bash
flux -n <ns> suspend kustomization frontend-sync
flux -n <ns> resume kustomization frontend-sync
```

## Common commands

Every `flux` command needs `-n <ns>`: nothing lives in `flux-system`.

```bash
flux -n <ns> check                                          # controllers healthy, version
flux -n <ns> get sources git                                # fetched revision of the branch
flux -n <ns> get kustomizations                             # Ready / revision per Kustomization
flux -n <ns> reconcile kustomization <ns> --with-source     # fetch the branch now and re-apply the root
flux -n <ns> reconcile kustomization cms-api-sync           # re-apply one app now
flux -n <ns> logs --kind=Kustomization --name=cms-api-sync
```

## Known limits

- The deploy key is each instance's only credential for the repository. If it's removed from
  GitHub, that instance stops fetching and stays on the last revision.
- Each instance's kustomize-controller has cluster-admin (Flux's default), so the namespace boundary
  is a convention kept by `APP_NAMESPACE`/`APP_ENV`, not enforced. Keep
  `<APP_NAMESPACE>-<APP_ENV>` equal to the namespace Flux was bootstrapped into.
- Flux's own Namespace object is in `gotk-components.yaml`: deleting an instance's root Kustomization
  with pruning on deletes the namespace and every app in it.
