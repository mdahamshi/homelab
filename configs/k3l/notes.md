# Host: k3.l — k3s Kubernetes cluster

Single-node k3s cluster, managed separately from the Docker hosts. Reachable via `ssh k3.l`.

## Cluster facts

- **k3s version**: v1.35.5+k3s1 (single control-plane node `k3s-base`, Ubuntu 26.04 LTS)
- **Container runtime**: containerd 2.2.3-k3s1
- **kubeconfig**: `/etc/rancher/k3s/k3s.yaml` (root-only; not committed — see redaction notes)
- **Server config**: `system/k3s-config.yaml` — only custom setting is `tls-san`
- **Helm**: no Helm CLI installed. Traefik is installed via k3s' bundled Helm controller
  (`HelmChart` in `/var/lib/rancher/k3s/server/manifests/traefik.yaml`); every other workload
  is applied with plain `kubectl apply` (no Helm annotations).

## Ingrees routing

- Ingress class: `traefik` (k3s bundled Traefik chart, `rancher/mirrored-library-traefik:3.6.13`)
- Traefik `LoadBalancer` service publishes to the node address (svclb daemonset).
- **No cert-manager** and no HTTPS on the ingress layer — Traefik's `websecure` entrypoint
  exists but no certificates resolver is configured; all ingresses are plain HTTP (`:80`).
- Public DNS for the `top-members` prod ingress also points a public domain at the node
  (sanitized in the manifests; presumably routed via the external WAF/Cloudflare layer).

## Workloads

| Namespace    | Workload            | Kind          | Image (sanitized)                     |
|--------------|---------------------|---------------|---------------------------------------|
| default      | jenkins-k3s-app     | Deployment    | ghcr.io/example-user/jenkins-k3s-pipeline:42 |
| default      | learn-jenkins-app   | Deployment    | ghcr.io/example-user/learn-jenkins-app:79     |
| staging      | learn-jenkins-app   | Deployment    | ghcr.io/example-user/learn-jenkins-app:79     |
| staging      | client / server     | Deployment    | ghcr.io/example-user/top-members-client:5 / -server:5 |
| staging      | postgres            | StatefulSet   | postgres:15                            |
| top-members  | client / server     | Deployment    | ghcr.io/example-user/top-members-client:15 / -server:15 |
| top-members  | postgres            | StatefulSet   | postgres:15                            |
| top-members  | db-seed             | Job (complete) | postgres:15                            |
| kube-system  | traefik, coredns, metrics-server, local-path-provisioner | — | k3s bundled |

Images are private GHCR packages (sanitized from the real account) with a pull secret
(`ghcr-secret` in `default`, type `kubernetes.io/dockerconfigjson`) — see Secret note.

## Storage

- Default StorageClass: `local-path` (`rancher.io/local-path`, `Delete`, `WaitForFirstConsumer`)
- Two 1Gi `ReadWriteOnce` PVCs (one per postgres StatefulSet) land on the node under
  `/var/lib/rancher/k3s/storage/`. See `system/storage.yaml`.

## Relation to the rest of the homelab

- **Separate from col.l**: Coolify does NOT deploy here. All k3s workloads use
  `kubectl apply`-ed manifests and GHCR images, and the images are built/published by
  Jenkins (the `jenkins-k3s-app` / `learn-jenkins-app` workloads are experiments with a
  Jenkins pipeline that builds images and deploys to k3s).
- Shares the same **TLS domain space** as the Docker hosts: `*.k3.example.home`
  in DNS (real cluster subdomain sanitized to `.k3.example.home`).
- Ingress traffic egresses via Traefik on the node IP; public exposure of the
  `top-members` app goes through the same external zone as the other hosts.

## Redaction notes

- Real GHCR owner + image names replaced with `ghcr.io/example-user/*`.
- Real cluster domain `*.k3.example.home` replaces the actual cluster subdomain;
  public host `*.example.com` replaces the public zone; node IP `10.0.x.x` replaces
  the real LAN IP.
- `metadata.uid`, `resourceVersion`, `creationTimestamp`, `status`, `managedFields`,
  `kubectl.kubernetes.io/last-applied-configuration` (and `restartedAt`) are stripped
  from all manifests — they contain live cluster info.
- **Secrets are deliberately omitted** (not even redacted copies), per house rules:
  - `default/ghcr-secret` — dockerconfigjson for pulling private GHCR images.
  - `staging/top-members-secrets` and `top-members/top-members-secrets` (Opaque, 6 keys) —
    `DATABASE_URL`, `POSTGRES_DB/USER/PASSWORD`, `MEMBERSHIP_PASSCODE`, `SESSION_SECRET`
    are consumed via `secretKeyRef` / `secretRef` in the server deployment, postgres
    StatefulSet, and db-seed job.
- `db-seed-data` ConfigMap originally contained real user accounts (real usernames,
  bcrypt password hashes, family member names). Replaced with placeholder data to show
  the schema while keeping the seed job realistic.
