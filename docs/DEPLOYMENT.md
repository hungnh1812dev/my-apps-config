# Deployment guide

From an empty VPS to cms-api, cms-admin and frontend live on HTTPS, then day-to-day operation.
Do the steps in order. Commands in `<angle brackets>` need your values.

Placeholders used throughout:

| Placeholder | Meaning | Example |
|---|---|---|
| `<domain>` | Bare domain (`APP_DOMAIN`) | `example.com` |
| `<ns>` | App namespace, `<APP_NAMESPACE>-<APP_ENV>` | `project-me-prod` |
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

Install the CLI at the version already committed in `gotk-components.yaml` (v2.9.5). Bootstrap
writes the components for the CLI's own version, so a newer CLI would upgrade Flux as a side
effect. Upgrade on purpose later with step 13.

```bash
curl -s https://fluxcd.io/install.sh | sudo FLUX_VERSION=2.9.5 bash
flux --version
flux check --pre
```

Before bootstrapping, create the `flux-system` namespace and the shared ConfigMap. The root
Kustomization reads it to name the app Kustomizations, and fails without it:

```bash
kubectl create namespace flux-system
cp templates/shared-configmap.example.yaml templates/shared-configmap.yaml
# Set APP_NAME, APP_NAMESPACE, APP_ENV, APP_DOMAIN, APP_TLS_CLUSTER_ISSUER. Keep metadata.name: shared-config.
kubectl apply --server-side -f templates/shared-configmap.yaml
```

Bootstrap. This installs the controllers, creates an SSH deploy key on the GitHub repository,
stores its private half in the `flux-system` Secret, and points Flux at `cluster/me`:

```bash
export GITHUB_TOKEN=<personal-access-token>
flux bootstrap github \
  --owner=hungnh1812dev \
  --repository=project-me-config \
  --branch=main \
  --path=cluster/me \
  --personal
```

- `--path=cluster/me` is required. Any other path rewrites `gotk-sync.yaml`, and the apps are
  no longer applied.
- If the generated files differ from the committed ones (for example, the `url` without `.git`),
  bootstrap commits the difference to `main`. Run `git pull` afterwards.
- The token is only used during bootstrap. Flux itself uses the deploy key, which is read-only.

The three app Kustomizations are created now and **fail until steps 6–8 are done** (missing
ConfigMap, namespace or Secret). That's expected. They retry every minute.

```bash
flux get kustomizations -n flux-system
```

## 6. Create the namespace

All three apps share one namespace, and Flux doesn't create it ([AUDIT.md](AUDIT.md), finding 8).
Use the same `APP_NAMESPACE` and `APP_ENV` values you'll put in the ConfigMaps:

```bash
kubectl create namespace <ns>
```

## 7. Apply the per-app ConfigMaps

These hold the `${APP_*}` values Flux substitutes into the manifests. They go in `flux-system`.
The shared one (`shared-config`) was applied in step 5. Every sync file reads it first, then its
own per-app ConfigMap.

```bash
for svc in cms-api cms-admin frontend; do
  cp templates/$svc-configmap.example.yaml templates/$svc-configmap.yaml
done
# Per-app copies: set APP_SERVICE_NAME (cms-api, cms-admin, frontend), APP_IMAGE_REPO,
#   APP_PORT (cms-api and frontend; "3000" for frontend), and metadata.name to
#   <APP_NAME>-<svc>-<APP_ENV>-config (e.g. project-me-cms-api-prod-config).
kubectl apply --server-side -f templates/cms-api-configmap.yaml
kubectl apply --server-side -f templates/cms-admin-configmap.yaml
kubectl apply --server-side -f templates/frontend-configmap.yaml
```

Check that the names match what the sync files read:

```bash
kubectl -n flux-system get configmap shared-config \
  project-me-cms-api-prod-config project-me-cms-admin-prod-config project-me-frontend-prod-config
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
flux reconcile kustomization flux-system --with-source
flux reconcile kustomization project-me-cms-api-sync-prod
flux reconcile kustomization project-me-cms-admin-sync-prod
flux reconcile kustomization project-me-frontend-sync-prod
flux get kustomizations -n flux-system       # all four READY True
```

cms-admin and frontend start with `APP_IMAGE_TAG: "dev"`. If there's no `dev` tag in the
registry, they stay not Ready until the app repository commits a real tag (step 11).

Check the workloads and certificates:

```bash
kubectl -n <ns> get pods,svc,ingress
kubectl -n <ns> get certificate              # one <name>-tls per service, READY True
```

Certificates can take a minute or two. Then check each site over HTTPS:

```bash
curl -I https://<domain>/api/health          # frontend: 200
curl -I https://admin.<domain>/healthz       # cms-admin: 200
curl -I https://api.<domain>/health          # cms-api: 200
curl -I http://<domain>                      # 301/308 redirect to https
```

## 11. Update image tags from the app repository

Normally this is automatic. The app repository's pipeline pushes an image, rewrites the
`APP_IMAGE_TAG:` line in `cluster/me/<svc>-sync.yaml`, and commits to `main`. Flux fetches
`main` every minute and rolls out the new tag. The rules the pipeline must follow are in
[components/configuration.md](components/configuration.md#image-tag-contract-with-the-app-repository).

To deploy a tag by hand, make the same change:

```bash
sed -i -E 's/^( *APP_IMAGE_TAG: ).*/\1"<tag>"/' cluster/me/cms-api-sync.yaml   # macOS: sed -i ''
git commit -am "deploy(cms-api): <tag>" && git push
flux reconcile kustomization flux-system --with-source      # optional: don't wait for the poll
kubectl -n <ns> rollout status deploy/<name>
```

## 12. Rollback

**Normal path:** revert the tag commit. Git stays the source of truth, and Flux rolls back on its
next poll.

```bash
git log --oneline -- cluster/me/cms-api-sync.yaml
git revert <commit> && git push
flux reconcile kustomization flux-system --with-source
```

**Emergency path:** roll back on the cluster first, then fix git. Suspend the app's Kustomization,
or Flux will reapply the bad tag within minutes:

```bash
flux suspend kustomization project-me-cms-api-sync-prod
kubectl -n <ns> rollout undo deploy/<name>
# ...revert the tag commit in git as above, then:
flux resume kustomization project-me-cms-api-sync-prod
```

**cms-api:** rolling back the image doesn't roll back database migrations. Only roll back past a
migration if it's backward-compatible ([components/cms-api.md](components/cms-api.md#known-limits)).

## 13. Upgrading Flux

Read the release notes for API changes first (<https://github.com/fluxcd/flux2/releases>). Then
install the new CLI and re-run the same bootstrap command as in step 5. Bootstrap regenerates
`gotk-components.yaml` for the new version, commits it, and the controllers upgrade themselves.

```bash
curl -s https://fluxcd.io/install.sh | sudo FLUX_VERSION=<new-version> bash   # no leading "v"
export GITHUB_TOKEN=<personal-access-token>
flux bootstrap github --owner=hungnh1812dev --repository=project-me-config \
  --branch=main --path=cluster/me --personal
flux check
git pull
```

To review the change before it's applied, write the files locally and open a PR instead:

```bash
flux install --export > cluster/me/flux-system/gotk-components.yaml
git diff --stat   # review, commit, push
```

## 14. Troubleshooting

| Symptom | Likely cause | Check / fix |
|---|---|---|
| `flux get sources git` not Ready, auth error | Deploy key removed from GitHub | Re-run the step 5 bootstrap to recreate it |
| `flux-system` Kustomization: path not found | Bootstrapped with a different `--path` | Re-run bootstrap with `--path=cluster/me` |
| App Kustomization: `ConfigMap ... not found` | Step 7 skipped or wrong name | `kubectl -n flux-system get cm`; names must be `shared-config` and `<APP_NAME>-<svc>-<APP_ENV>-config` |
| App Kustomization: `namespaces "<ns>" not found` | Step 6 skipped, or namespace doesn't match the ConfigMap | `kubectl get ns`; compare with `APP_NAMESPACE`-`APP_ENV` |
| Ingress rejected: host `api.` / `admin.` / `APP_DOMAIN-is-not-set` | `APP_DOMAIN` missing from the ConfigMap | Set it, re-apply, reconcile |
| Pod `CreateContainerConfigError`, Secret not found | Step 8 skipped or Secret name/namespace wrong | `kubectl -n <ns> get secret`; name must be `<name>-secrets` |
| Pod `ImagePullBackOff` | Tag doesn't exist (e.g. `dev`), or private image without a pull secret | `kubectl -n <ns> describe pod`; commit a real tag or do step 9 |
| cms-api pod stuck in `Init:CrashLoopBackOff` | `prisma migrate deploy` failed (DB unreachable or bad credentials) | `kubectl -n <ns> logs <pod> -c init` |
| cms-admin crash-loops with a `chown`/`setgid` error | nginx needs a capability the Deployment dropped | Remove the `capabilities` block ([AUDIT.md](AUDIT.md), finding 4) |
| Certificate not Ready | DNS not resolving yet, or port 80 blocked | `kubectl -n <ns> describe challenge`; check step 4 and the firewall |
| Browser: cms-admin API calls blocked (CORS) | `CORS_ORIGINS` doesn't include `https://admin.<domain>` | Fix the cms-api Secret, re-apply, `rollout restart` |
| Secret changed but the app still uses old values | Pods read `envFrom` only at start | `kubectl -n <ns> rollout restart deploy/<name>` |
| ConfigMap changed but nothing happened | Flux hasn't re-rendered yet | `flux reconcile kustomization project-me-<svc>-sync-prod` |
| Manual `kubectl edit` is reverted | Flux re-applies git every 3m | Change it in git, or `flux suspend` first |
| App Kustomization times out (5m) while pods look fine | A resource isn't healthy (`wait: true`) | `flux get kustomizations`; `kubectl -n <ns> get events --sort-by=.lastTimestamp` |
