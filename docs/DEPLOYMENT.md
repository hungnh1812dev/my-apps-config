# Deployment guide

From an empty VPS to cms-api, cms-admin and frontend live on HTTPS, then day-to-day operation.
Do the steps in order. Commands in `<angle brackets>` need your values.

Placeholders used throughout:

| Placeholder | Meaning | Example |
|---|---|---|
| `<domain>` | Bare domain (`APP_DOMAIN`) | `example.com` |
| `<ns>` | Environment namespace (Flux + apps), `<APP_NAMESPACE>-<APP_ENV>` | `project-me-prod`, `project-me-staging` |
| `<env>` / `<branch>` | Environment and its branch | `staging` / `staging`, `prod` / `main` |
| `<name>` | `<APP_SERVICE_NAME>-<APP_ENV>` for one service | `cms-api-prod` |
| `<vps-ip>` | Public IP of the server | |

See [components/configuration.md](components/configuration.md) for how the names are built.

## 1. Prerequisites

- A Linux VPS, **amd64** (image tags end in `-amd64`), 2 GB RAM or more, with ports 80 and 443
  open to the internet and SSH for you.
- A domain whose DNS you control.
- A Postgres database reachable from the VPS (cms-api's `DB_HOST`).
- Images for all three services pushed to GHCR by the app repository.
- A GitHub personal access token for `flux bootstrap` (step 5). A classic token needs the `repo`
  scope. A fine-grained token needs **Administration: read/write** (to add the deploy key) and
  **Contents: read/write** on `project-me-config`.
- On your workstation: `git`, `kubectl`, and a clone of this repository. The filled-in templates
  stay on your workstation and are gitignored.

## 2. Install k3s

On the VPS:

```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes          # STATUS Ready
```

k3s includes Traefik (the Ingress controller used here, class `traefik`) with its
`traefik.io/v1alpha1` CRDs.

To use `kubectl` from your workstation without exposing the API port, copy the kubeconfig and
tunnel port 6443 over SSH:

```bash
scp root@<vps-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/project-me.yaml   # server: https://127.0.0.1:6443
ssh -N -L 6443:127.0.0.1:6443 root@<vps-ip> &                      # keep this running
export KUBECONFIG=~/.kube/project-me.yaml
kubectl get nodes
```

All later commands assume `kubectl` reaches the cluster this way (or run them on the VPS).

## 3. cert-manager and a ClusterIssuer

Install cert-manager. Pick the latest version from
<https://github.com/cert-manager/cert-manager/releases>:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/<cert-manager-version>/cert-manager.yaml
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
```

Create a Let's Encrypt ClusterIssuer that solves HTTP-01 challenges through Traefik:

```bash
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <you@example.com>
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
EOF
kubectl get clusterissuer letsencrypt-prod      # READY True
```

Its name (`letsencrypt-prod`) goes into `APP_TLS_CLUSTER_ISSUER` in step 7. While testing, you can
use a second issuer with the staging server (`https://acme-staging-v02.api.letsencrypt.org/directory`)
to avoid Let's Encrypt rate limits.

## 4. DNS

Create three records pointing at the VPS:

| Name | Type | Value | Serves |
|---|---|---|---|
| `<domain>` | A | `<vps-ip>` | frontend |
| `admin.<domain>` | A | `<vps-ip>` | cms-admin |
| `api.<domain>` | A | `<vps-ip>` | cms-api |

Check them before step 10, because certificates can't be issued until they resolve:

```bash
dig +short <domain> admin.<domain> api.<domain>
```

## 5. Install the Flux CLI and bootstrap

Each environment is its own Flux instance, installed into the environment's namespace and watching
only that namespace. Several instances (other environments, other projects) can share one k3s. See
[components/flux-system.md](components/flux-system.md) for why each flag is there.

| Environment | `<ns>` | `--branch` | `--path` |
|---|---|---|---|
| staging | `project-me-staging` | `staging` | `cluster/me/staging` |
| prod | `project-me-prod` | `main` | `cluster/me/prod` |

Install the CLI at the version already committed in `gotk-components.yaml` (v2.9.5), and use the
same version for every instance on the cluster: the CRDs are shared. Bootstrap writes the
components for the CLI's own version, so a newer CLI would upgrade Flux as a side effect.

```bash
curl -s https://fluxcd.io/install.sh | sudo FLUX_VERSION=2.9.5 bash
flux --version
flux check --pre
```

The branch must exist and contain `cluster/me/<env>/` before you bootstrap. Then, per environment:

```bash
export GITHUB_TOKEN=<personal-access-token>
flux bootstrap github \
  --owner=hungnh1812dev \
  --repository=project-me-config \
  --branch=<branch> \
  --path=cluster/me/<env> \
  --namespace=<ns> \
  --watch-all-namespaces=false \
  --network-policy=false \
  --components=source-controller,kustomize-controller \
  --personal
```

- Keep every flag on re-runs (upgrades, key rotation). Bootstrap regenerates `gotk-*.yaml` from the
  flags, so a missing flag silently changes the install.
- Bootstrap creates `<ns>`, a deploy key per instance on the GitHub repository, and the `flux-system`
  Secret in `<ns>`. It commits `gotk-components.yaml` and `gotk-sync.yaml` to `<branch>` if they
  differ from what's there. Run `git pull` afterwards.
- The token is only used during bootstrap. Flux itself uses the deploy key, which is read-only.

The three app Kustomizations are created now and **fail until steps 7–8 are done** (missing
ConfigMap or Secret). That's expected. They retry every minute.

```bash
flux -n <ns> get kustomizations
```

## 6. Check the namespace

Bootstrap created `<ns>`, and Flux, the ConfigMaps, the Secrets and all three apps live there.
`APP_NAMESPACE`-`APP_ENV` in step 7 must produce exactly this name.

```bash
kubectl get namespace <ns>
kubectl -n <ns> get deploy          # source-controller, kustomize-controller
```

## 7. Apply the ConfigMaps

These hold the `${APP_*}` values Flux substitutes into the manifests. They go in `<ns>`, under fixed
names: `shared-config`, then one per app (`cms-api-config`, `cms-admin-config`, `frontend-config`).
Every sync file reads the shared one first, then its own.

```bash
cp templates/shared-configmap.example.yaml templates/shared-configmap-<env>.yaml
for svc in cms-api cms-admin frontend; do
  cp templates/$svc-configmap.example.yaml templates/$svc-configmap-<env>.yaml
done
# Set metadata.namespace to <ns> in all four.
# Shared: APP_NAME, APP_NAMESPACE + APP_ENV (together = <ns>), APP_DOMAIN (different per environment,
#   e.g. staging.<domain>), APP_TLS_CLUSTER_ISSUER.
# Per app: APP_SERVICE_NAME (cms-api, cms-admin, frontend), APP_IMAGE_REPO,
#   APP_PORT (cms-api and frontend; "3000" for frontend).
for f in shared cms-api cms-admin frontend; do
  kubectl apply --server-side -f templates/$f-configmap-<env>.yaml
done
kubectl -n <ns> get configmap shared-config cms-api-config cms-admin-config frontend-config
```

Every variable is described in [components/configuration.md](components/configuration.md).

## 8. Apply the two Secrets

cms-api and frontend read their runtime settings from a Secret in `<ns>`. cms-admin has none.

```bash
cp templates/cms-api-secret.example.yaml  templates/cms-api-secret.yaml
cp templates/frontend-secret.example.yaml templates/frontend-secret.yaml
# Edit both: set metadata.name to <name>-secrets for that service (cms-api-prod-secrets,
# frontend-prod-secrets), metadata.namespace
# to <ns>, and fill in the values.
kubectl apply --server-side -f templates/cms-api-secret.yaml
kubectl apply --server-side -f templates/frontend-secret.yaml
```

Use `--server-side`. A plain `kubectl apply` also stores every value in plain text in the
`last-applied-configuration` annotation.

Values that link the services together:

- cms-api `CORS_ORIGINS` must include `https://admin.<domain>`.
- frontend `CMS_API_URL` / `GRAPHQL_URL` should use the in-cluster cms-api Service
  (`http://cms-api-<APP_ENV>.<ns>.svc.cluster.local:<APP_PORT>`).
- frontend `REVALIDATE_SECRET` must match the value cms-api sends.

Keys per service: [cms-api](components/cms-api.md#configuration),
[frontend](components/frontend.md#configuration).

## 9. (Optional) GHCR pull secret

Skip this if the GHCR packages are public. For private packages, create a token with
`read:packages` and attach it to the namespace's default service account, so no manifest changes:

```bash
kubectl -n <ns> create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username=<github-user> \
  --docker-password=<token-with-read:packages>
kubectl -n <ns> patch serviceaccount default \
  -p '{"imagePullSecrets":[{"name":"ghcr-pull"}]}'
```

Pods created after the patch use it. Delete any pod already in `ImagePullBackOff` so it's recreated.

## 10. Reconcile and verify

Trigger a reconcile instead of waiting for the next retry:

```bash
flux -n <ns> reconcile kustomization <ns> --with-source
flux -n <ns> reconcile kustomization cms-api-sync
flux -n <ns> reconcile kustomization cms-admin-sync
flux -n <ns> reconcile kustomization frontend-sync
flux -n <ns> get kustomizations              # all four READY True
```

If a tag in `cluster/me/<env>/<svc>-sync-overlay.yaml` doesn't exist in the registry (e.g. `dev`),
that app stays not Ready until the app repository commits a real tag (step 11).

Check the workloads and certificates:

```bash
kubectl -n <ns> get pods,svc,ingress
kubectl -n <ns> get certificate              # one <name>-tls per service, READY True
```

Certificates can take a minute or two. Then check each site over HTTPS:

```bash
curl -I https://<domain>/api/health/ready    # frontend: 200
curl -I https://admin.<domain>/health/ready  # cms-admin: 200
curl -I https://api.<domain>/health/ready    # cms-api: 200
curl -I http://<domain>                      # 301/308 redirect to https
```

## 11. Update image tags from the app repository

Normally this is automatic. The app repository's pipeline pushes an image, rewrites the
`APP_IMAGE_TAG:` line in `cluster/me/<env>/<svc>-sync-overlay.yaml`, and commits it to that
environment's branch (`staging` or `main`). Flux fetches the branch every minute and rolls out the
new tag. The rules the pipeline must follow are in
[components/configuration.md](components/configuration.md#image-tag-contract-with-the-app-repository).

To deploy a tag by hand, make the same change on the environment's branch:

```bash
sed -i -E 's/^( *APP_IMAGE_TAG: ).*/\1"<tag>"/' cluster/me/<env>/cms-api-sync-overlay.yaml   # macOS: sed -i ''
git commit -am "deploy(cms-api, <env>): <tag>" && git push
flux -n <ns> reconcile kustomization <ns> --with-source      # optional: don't wait for the poll
kubectl -n <ns> rollout status deploy/<name>
```

## 12. Rollback

**Normal path:** revert the tag commit on the environment's branch. Git stays the source of truth,
and Flux rolls back on its next poll.

```bash
git log --oneline -- cluster/me/<env>/cms-api-sync-overlay.yaml
git revert <commit> && git push
flux -n <ns> reconcile kustomization <ns> --with-source
```

**Emergency path:** roll back on the cluster first, then fix git. Suspend the app's Kustomization,
or Flux will reapply the bad tag within minutes:

```bash
flux -n <ns> suspend kustomization cms-api-sync
kubectl -n <ns> rollout undo deploy/<name>
# ...revert the tag commit in git as above, then:
flux -n <ns> resume kustomization cms-api-sync
```

**cms-api:** rolling back the image doesn't roll back database migrations. Only roll back past a
migration if it's backward-compatible ([components/cms-api.md](components/cms-api.md#known-limits)).

## 13. Upgrading Flux

The CRDs are shared by every Flux instance on the cluster, so upgrade all of them together, other
projects included. Read the release notes for API changes first
(<https://github.com/fluxcd/flux2/releases>). Then install the new CLI and re-run the step 5 bootstrap
command, with the same flags, for each instance. Bootstrap regenerates `gotk-components.yaml` for
the new version, commits it, and the controllers upgrade themselves.

```bash
curl -s https://fluxcd.io/install.sh | sudo FLUX_VERSION=<new-version> bash   # no leading "v"
# re-run the step 5 bootstrap for staging, then prod, then every other instance on the cluster
flux -n <ns> check
git pull
```

Upgrade staging first and merge the regenerated file forward only after it's healthy. The prod
instance's file is on `main` and is regenerated by the prod bootstrap, not by the merge.

## 14. Troubleshooting

| Symptom | Likely cause | Check / fix |
|---|---|---|
| `flux -n <ns> get sources git` not Ready, auth error | Deploy key removed from GitHub | Re-run the step 5 bootstrap to recreate it |
| Root Kustomization: path not found | Bootstrapped with a different `--path`, or the branch lacks `cluster/me/<env>` | Re-run bootstrap with the table's `--branch`/`--path` |
| App Kustomization: `GitRepository ... not found` | `sourceRef` patch in `cluster/me/<env>/kustomization.yaml` doesn't match `<ns>` | The GitRepository is named after the bootstrap namespace |
| App Kustomization: `ConfigMap ... not found` | Step 7 skipped, wrong name or wrong namespace | `kubectl -n <ns> get cm`; names must be `shared-config` and `<svc>-config` |
| Resources go to the wrong namespace | `APP_NAMESPACE`-`APP_ENV` doesn't equal `<ns>` | Fix `shared-config`, re-apply, reconcile |
| Objects reconciled twice / status flapping | Another Flux instance watches all namespaces | `kubectl get deploy -A -l app.kubernetes.io/part-of=flux -o yaml \| grep watch-all`; every instance needs `--watch-all-namespaces=false` |
| Sites time out, pods Ready | A Flux NetworkPolicy in `<ns>` blocks Traefik | `kubectl -n <ns> get networkpolicy`; bootstrap with `--network-policy=false` |
| Ingress rejected: host `api.` / `admin.` / `APP_DOMAIN-is-not-set` | `APP_DOMAIN` missing from the ConfigMap | Set it, re-apply, reconcile |
| Two environments fight over one host | Same `APP_DOMAIN` in staging and prod | Give staging its own domain |
| Pod `CreateContainerConfigError`, Secret not found | Step 8 skipped or Secret name/namespace wrong | `kubectl -n <ns> get secret`; name must be `<name>-secrets` |
| Pod `ImagePullBackOff` | Tag doesn't exist (e.g. `dev`), or private image without a pull secret | `kubectl -n <ns> describe pod`; commit a real tag or do step 9 |
| cms-api pod stuck in `Init:CrashLoopBackOff` | `prisma migrate deploy` failed (DB unreachable or bad credentials) | `kubectl -n <ns> logs <pod> -c init` |
| cms-admin crash-loops with a `chown`/`setgid` error | nginx needs a capability the Deployment dropped | Remove the `capabilities` block ([AUDIT.md](AUDIT.md), finding 4) |
| Certificate not Ready | DNS not resolving yet, or port 80 blocked | `kubectl -n <ns> describe challenge`; check step 4 and the firewall |
| Browser: cms-admin API calls blocked (CORS) | `CORS_ORIGINS` doesn't include `https://admin.<domain>` | Fix the cms-api Secret, re-apply, `rollout restart` |
| Secret changed but the app still uses old values | Pods read `envFrom` only at start | `kubectl -n <ns> rollout restart deploy/<name>` |
| ConfigMap changed but nothing happened | Flux hasn't re-rendered yet | `flux -n <ns> reconcile kustomization <svc>-sync` |
| Manual `kubectl edit` is reverted | Flux re-applies git every 3m | Change it in git, or `flux suspend` first |
| App Kustomization times out (5m) while pods look fine | A resource isn't healthy (`wait: true`) | `flux -n <ns> get kustomizations`; `kubectl -n <ns> get events --sort-by=.lastTimestamp` |

## 15. Removing one instance

Never run `flux uninstall` on a shared cluster: it deletes the CRDs, and with them every other
instance's Flux objects. To remove one environment and everything it deployed:

```bash
flux -n <ns> suspend kustomization --all
kubectl -n <ns> delete kustomizations.kustomize.toolkit.fluxcd.io --all   # suspended: nothing is pruned
kubectl -n <ns> delete gitrepositories.source.toolkit.fluxcd.io --all
kubectl delete clusterrolebinding,clusterrole -l app.kubernetes.io/instance=<ns>
kubectl delete namespace <ns>                                             # the apps go with it
```

Then delete the instance's deploy key under the repository's Settings → Deploy keys on GitHub.
To keep the apps running and only remove Flux, delete the Flux Deployments instead of the namespace:
`kubectl -n <ns> delete deploy -l app.kubernetes.io/part-of=flux`.

## 16. Migrating from the single `flux-system` instance

Before this layout, prod ran one Flux in `flux-system` (path `./cluster/me`, branch `main`, watching
all namespaces), and staging was bootstrapped the same way on the `staging` branch. Move each to its
own instance **before** the new layout reaches `main` or `staging`: the old root Kustomization reads
`./cluster/me`, which no longer has a `kustomization.yaml`, so Flux would generate one from every file
below it, including the other environment's `flux-system/`.

1. **Check which branch the cluster's `flux-system` follows.** If both environments share one k3s
   and staging was bootstrapped into `flux-system`, the prod cluster now follows `staging`:

   ```bash
   kubectl -n flux-system get gitrepository flux-system -o jsonpath='{.spec.ref.branch}{"\n"}'
   kubectl -n flux-system get kustomizations
   ```

   Do steps 2–3 before bootstrapping. While the old instance runs, its source-controller (watching
   all namespaces) also serves the new GitRepository, and the new root Kustomization fails with
   `failed to download archive: GET http://source-controller.flux-system.svc... connection refused`.

2. **Freeze the old instance.** Nothing is deleted from the apps while it's suspended:

   ```bash
   flux -n flux-system suspend kustomization --all
   ```

3. **Remove the old instance, keeping the apps and the CRDs:**

   ```bash
   kubectl -n flux-system delete kustomizations.kustomize.toolkit.fluxcd.io --all   # suspended: nothing is pruned
   kubectl -n flux-system delete gitrepositories.source.toolkit.fluxcd.io --all
   kubectl delete clusterrolebinding,clusterrole -l app.kubernetes.io/instance=flux-system
   kubectl delete namespace flux-system
   ```

   Delete the old deploy keys on GitHub (Settings → Deploy keys).

4. **Push the new layout** to `staging` and `main` (merge `develop`). Nothing reconciles it yet.

5. **Bootstrap each environment** (step 5). The existing namespace `project-me-<env>` is adopted,
   and the apps keep their names (`<svc>-<env>`), so running pods are adopted, not recreated.

6. **Recreate the ConfigMaps in `<ns>`** under the new names (step 7). Values are unchanged; only the
   name and namespace differ (`project-me-cms-api-prod-config` in `flux-system` becomes
   `cms-api-config` in `project-me-prod`). Secrets stay where they are.

7. **Verify** (step 10) for each environment.
