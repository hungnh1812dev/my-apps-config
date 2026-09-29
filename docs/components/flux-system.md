# flux-system: bootstrap and sync

How Flux gets from this Git repository to the three apps on the cluster. Everything here lives in
the `flux-system` namespace.

## Resources

| Resource | Name | Defined in | What it does |
|---|---|---|---|
| Flux controllers | source-, kustomize-, helm-, notification-controller | `cluster/me/flux-system/gotk-components.yaml` | The Flux v2.9.5 install (the default components). Generated, don't edit |
| Secret | `flux-system` | created by `flux bootstrap` (not in git) | SSH deploy key for GitHub |
| GitRepository | `flux-system` | `cluster/me/flux-system/gotk-sync.yaml` | Polls `main` every 1m |
| Kustomization | `flux-system` | `cluster/me/flux-system/gotk-sync.yaml`, patched by `flux-system/kustomization.yaml` | The root: applies `./cluster/me` every 10m, with `${APP_NAME}`/`${APP_ENV}` from `shared-config` |
| Kustomization | `project-me-cms-api-sync-prod` | `cluster/me/cms-api-sync.yaml` | Applies `./apps/cms-api` |
| Kustomization | `project-me-cms-admin-sync-prod` | `cluster/me/cms-admin-sync.yaml` | Applies `./apps/cms-admin` |
| Kustomization | `project-me-frontend-sync-prod` | `cluster/me/frontend-sync.yaml` | Applies `./apps/frontend` |

## How it fits together

```
GitHub: hungnh1812dev/project-me-config (main)
  │  ssh, deploy key in Secret flux-system
  ▼
GitRepository flux-system            (interval 1m)
  │
  ▼
Kustomization flux-system            path ./cluster/me (kustomization.yaml)
  ├── flux-system/                   gotk-components.yaml + gotk-sync.yaml (Flux manages itself)
  ├── cms-api-sync.yaml   ──► Kustomization project-me-cms-api-sync-prod   ──► apps/cms-api
  ├── cms-admin-sync.yaml ──► Kustomization project-me-cms-admin-sync-prod ──► apps/cms-admin
  └── frontend-sync.yaml  ──► Kustomization project-me-frontend-sync-prod  ──► apps/frontend
```

The root Kustomization only creates the three app Kustomizations. It reads `shared-config` to fill
`${APP_NAME}` and `${APP_ENV}` in their names and in the per-app ConfigMap each one reads (a patch in
`cluster/me/flux-system/kustomization.yaml`; Flux's own manifests are excluded). The names above are
what prod gets with `APP_NAME: project-me` and `APP_ENV: prod`. Each app Kustomization then
renders its `apps/<svc>` directory, substitutes `${APP_*}` from the shared and per-app ConfigMaps (see
[configuration.md](configuration.md)), and applies the result into the app namespace.

All four Kustomizations read the same GitRepository, so one commit updates everything. When the
GitRepository fetches a new revision, kustomize-controller reconciles right away. It doesn't wait
for the 10m or 3m interval, so a commit usually reaches the cluster within a minute or two.

## Bootstrap

`flux bootstrap github` installs the controllers, creates the deploy key and the `flux-system`
Secret, and commits `gotk-components.yaml`, `gotk-sync.yaml` and `flux-system/kustomization.yaml`
to `--path`. The files are already in this repository, so bootstrap only commits if they differ.

Always bootstrap with `--path=cluster/me`. `gotk-sync.yaml` is generated from the flags, and
any other path overwrites the root Kustomization's `spec.path`, so the apps are no longer applied.
The full command is in [DEPLOYMENT.md](../DEPLOYMENT.md).

`shared-config` must exist before bootstrap. Without it the root Kustomization fails, and with it
Flux's own updates.

Don't edit the `gotk-*.yaml` files by hand. `flux-system/kustomization.yaml` is the exception:
bootstrap keeps it, and it holds the substitution patch. Change them by re-running `flux bootstrap` (or
`flux install --export` for an upgrade; see the Upgrading Flux section in
[DEPLOYMENT.md](../DEPLOYMENT.md)).

## App Kustomization settings

All three sync files share these settings:

| Field | Value | Why |
|---|---|---|
| `interval` | `3m` | Re-applies the rendered manifests, which also reverts manual edits on the cluster |
| `retryInterval` | `1m` | A failed apply is retried sooner than the next interval |
| `prune` | `true` | A resource removed from `apps/<svc>` is deleted from the cluster |
| `wait` | `true` | Ready only once every applied resource is healthy (Deployment rolled out, etc.) |
| `timeout` | `5m` | How long `wait` waits before the reconcile is marked failed |
| `postBuild.substituteFrom` | `shared-config`, then `${APP_NAME}-<svc>-${APP_ENV}-config` | The `${APP_*}` values: shared first, per-app second (per-app wins on a clash) |
| `postBuild.substitute` | `APP_IMAGE_TAG` | The image tag, committed here by the app repository |

There's no `dependsOn` between apps. The frontend tolerates cms-api being down, and a `dependsOn`
would block frontend deploys whenever cms-api isn't Ready ([AUDIT.md](../AUDIT.md), finding 12).

## Pruning

Both levels have `prune: true`, so deletions in git propagate:

- Removing a file from `apps/<svc>/kustomization.yaml` deletes that resource.
- Removing a `*-sync.yaml` from `cluster/me/kustomization.yaml` deletes that app's
  Kustomization, and with it every resource the app deployed.

To stop an app from changing without deleting it, suspend it instead:

```bash
flux suspend kustomization project-me-frontend-sync-prod
flux resume kustomization project-me-frontend-sync-prod
```

## Common commands

```bash
flux check                                                  # controllers healthy, version
flux get sources git -n flux-system                         # fetched revision of main
flux get kustomizations -n flux-system                      # Ready / revision per Kustomization
flux reconcile kustomization flux-system --with-source      # fetch main now and re-apply the root
flux reconcile kustomization project-me-cms-api-sync-prod      # re-apply one app now
flux logs --kind=Kustomization --name=project-me-cms-api-sync-prod -n flux-system
```

## Known limits

- The deploy key is the cluster's only credential for the repository. If it's removed from GitHub,
  the GitRepository stops fetching and the cluster stays on the last revision.
- Flux applies whatever is on `main`. There's no staging branch or promotion step.
