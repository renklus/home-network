# CLAUDE.md
## What this repo is
GitOps configuration for a home-lab Kubernetes setup, deployed by Argo CD from `https://github.com/renklus/home-network` at `HEAD` of `main`. There is no build, lint or test tooling: a change takes effect when it is pushed and Argo CD syncs it (all apps use automated sync with `prune` and `selfHeal`). Validate edits by reasoning about the rendered manifests; `kubectl`/`argocd` access is not assumed.

## Active vs. legacy directories
- `k8s-rancher/apps/` — **active**. The management ("rancher") cluster: Argo CD, Rancher, cert-manager, MetalLB, HAProxy, CoreDNS customisation.
- `k8s-prod/apps/` — **active**. Workloads for the Rancher-provisioned `prod` cluster (immich, CNPG Postgres, csi-nfs, MetalLB, system config).
- `k8s/`, `k8s-dmz/`, `k8s-old1/` — older cluster setups, unmaintained and listed in Renovate's `ignorePaths`. Don't use them as patterns for new work, and don't add Renovate rules for them.
- `images/` — Dockerfiles for small custom images, published by `.github/workflows/` (note the workflows reference `./container/images/...` paths and the `master` branch, which don't match the current layout).
- `docs/cluster-bootstrap.md` — manual steps outside Argo CD: workstation setup, k3s install, LAN DNS/NAT and Cloudflare, the Argo CD bootstrap install, node joins.
- `docs/truenas-backup.md` — TrueNAS dataset/NFS share behind the prod storage classes and its rsync backup to a Synology, with open steps (start here when asked what to do next on backups). `docs/rsync.md` is the general TrueNAS → Synology rsync (module mode) and snapshot setup.
- `docs/k8s-rancher-cluster-connection.md` — how Argo CD on the rancher cluster authenticates to Rancher-managed clusters via the Rancher auth proxy.

## Working in this repository
- Use 'k8s01d' instead of 'kubectl'.
- "Don't run cluster commands, tell me what to run"
- Don't commit changes. Work in the worktree as usual, but once I am happy with the change don't push or merge to master but build me a git diff command (git -C .claude/worktrees/<worktree-name> diff -- . | git apply) that applies the changes to the main working directory. Do not make tool calls just to build this command.
- Avoid unnecessary CLI tool calls. Use on board tools where feasible. For example prefer Read over `cat`-ing or `grep`-ing files.
- Value following best practices, a focus on maintainability and simplicity.

## Architecture
**App-of-apps chain.** Argo CD is installed manually during cluster bootstrap, then takes over its own management:
1. `k8s-rancher/apps/app-of-apps.yaml` defines the `rancher-cluster` AppProject and the root `app-of-apps` Application, which syncs the whole `k8s-rancher/apps` directory (non-recursive — only top-level files).
2. Each top-level `*.yaml` there is either an Argo CD `Application` or plain resources applied directly by the root app (e.g. `coredns.yaml`, `traefik.yaml`).
3. `app-of-prod.yaml` points at `k8s-prod/apps`, whose Applications use project `prod-cluster` and destination `name: production`.
4. `argocd.yaml` must keep `metadata.name: argo-cd` so it adopts the bootstrap install.
5. `k8s-rancher/apps/argocd/values.yaml` holds the Argo CD values shared by the bootstrap install (command in `docs/cluster-bootstrap.md`) and `argocd.yaml`. It includes the Application health check that makes sync waves between Applications wait (haproxy 0 → cert-manager 1 → apps with production certificates 2). The staging `demo-certificate` in cert-manager acts as a canary for the public HTTP-01 path.

**Per-app file layout.** An app `foo` is `foo.yaml` (the Application) plus an optional `foo/` subdirectory with extra manifests. Helm-based apps use multiple `sources`: the chart first, then `path: <cluster>/apps/foo` from this repo. Resources that depend on a chart's CRDs (e.g. MetalLB `IPAddressPool`, cert-manager `ClusterIssuer`) must live in that app's subdirectory, not at the top level, so they sync with the chart.

**Conventions every Application follows:**
- `syncOptions: CreateNamespace=false` and `ServerSideApply=true`. Namespaces are declared explicitly in `foo/namespaces.yaml` (or inline). One comment explains why.
- Explicit `project` is required: the `default` AppProject is locked down in `k8s-rancher/apps/argocd/projects.yaml`. `prod-cluster` blocks `Application` resources and the `argocd` namespace.
- Finalizer choice is deliberate: `resources-finalizer.argocd.argoproj.io` cascades deletion; it is intentionally omitted on `argo-cd`, `app-of-apps`, `app-of-prod`, `no-auto-delete` (which holds PVCs like the immich media claim) and `postgres` (the CNPG operator; deleting its CRDs would delete every database).
- Helm config uses `helm.parameters` for scalar settings and `helm.valuesObject` for image pins.

**Image pinning and Renovate.** Images are pinned as `repository: '...'` + `tag: 'vX.Y.Z@sha256:...'` with single quotes, and chart versions in `targetRevision: 'x.y.z'`

**Networking / TLS on the rancher cluster:**
- MetalLB pool `10.1.0.235-10.1.0.245`; using fixed IPs should be avoided where possible but it is possible via `metallb.universe.tf/loadBalancerIPs` at the top of the pool (traefik `.244`, external DNS `.245`). Prod uses `10.1.1.x`.
- CoreDNS `k8s_external` serves `<svc>.<ns>.rancher.k8s.renklus.ch` for LoadBalancer services; `coredns.yaml` also holds name rewrites (e.g. the `argo.` alias).
- Ingress is k3s Traefik (`ingressClassName: traefik`). Ingress hosts follow `<name>.ingress.rancher.k8s.renklus.ch`; a regex rewrite in `coredns.yaml` resolves all of them to Traefik, also from inside the cluster.
- cert-manager ClusterIssuers `letsencrypt-staging-shortlived` and `letsencrypt-shortlived` (prod) use HTTP-01. External port 80 reaches HAProxy (`haproxy.yaml`), which only forwards `/.well-known/acme-challenge/` and routes by Host to Traefik or to external dev hosts; cert-manager's self-check uses HAProxy as its `http_proxy`. Hosts under `k8s.renklus.ch` are covered by the public Cloudflare wildcard `*.k8s.renklus.ch` and HAProxy's suffix rules; any other new ACME host needs a HAProxy rule and a Cloudflare DNS record (outside this repo).
- Prod has its own cert-manager (`k8s-prod/apps/cert-manager.yaml`) with the same ClusterIssuers. HAProxy routes every host under `.prod.k8s.renklus.ch` to the prod Traefik, and prod cert-manager's self-check uses the rancher HAProxy as its `http_proxy`. Prod ingress hosts follow `<name>.ingress.prod.k8s.renklus.ch` (CoreDNS rewrite in `k8s-prod/apps/system-config/coredns-extension.yaml`).
- Rancher uses `ingress.tls.source=secret` with the `tls-rancher-ingress` Certificate in `k8s-rancher/apps/rancher/certificate.yaml`; its `dnsNames` must match the chart's `hostname`.

**Rancher-managed clusters.** `k8s-rancher/apps/rancher/cluster-*.yaml` define `provisioning.cattle.io/v1` Clusters (k3s) in `fleet-default`. Argo CD reaches them through the Rancher auth proxy: the Kyverno `GeneratingPolicy` in `k8s-rancher/apps/kyverno/argocd-clusters.yaml` turns Rancher's `fleet-default/<cluster>-kubeconfig` Secret into an Argo CD cluster Secret named after the Rancher cluster (e.g. `production`). No cluster IDs or tokens belong in git; reinstalling nodes or adding a cluster needs no Argo CD change.
