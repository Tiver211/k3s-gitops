# k3s-gitops

[По-русски](README.md)

The state of a home k3s cluster, kept in git. ArgoCD watches this repository and brings the cluster in line with what is described here. There is no application source code — only Helm wrappers and ArgoCD manifests.

The cluster replaces a home setup previously running on Docker Compose, so some services have already moved over and some have not.

## Hardware

Two Proxmox hosts running five k3s virtual machines: one control plane and four worker nodes, two of which are dedicated to storage. Debian 13, containerd. There is no HA — the control plane is a single node. That is a deliberate trade-off until a second set of hardware shows up, not an oversight.

## How it is put together

Exactly one file is applied to the cluster by hand: `bootstrap/root.yaml`. It is an ArgoCD Application that recursively watches the `bootstrap/` directory and creates every other Application from it. Everything else follows on its own: a file appears in git and the component is deployed, a file is removed and the component is gone.

Startup order matters, so infrastructure components carry sync waves. ArgoCD waits for a wave to become healthy before it starts the next one:

- 0 — sealed-secrets, ArgoCD
- 1 — cert-manager
- 2 — database operators: CloudNativePG, MariaDB, Redis
- 3 — Longhorn, csi-driver-nfs
- 4 — MetalLB
- 5 — ingress-nginx
- 6 — ingress for ArgoCD, kube-prometheus-stack, Loki
- 7 — Alloy
- 8, 9 — applications

Applications are wired in through two paths. Those that were adopted from manual deployment and need non-default sync settings have an explicit Application in `bootstrap/apps/`. Everything else is picked up by an ApplicationSet: create an `apps/<name>/` directory and it deploys itself into the `<name>` namespace.

## Privilege boundary

Two AppProjects. `infra` may create cluster-scoped resources — ClusterRole, CRDs, StorageClass. `apps` may not create anything outside its own namespace. If an application chart tries to create a ClusterRole, the sync fails with a clear error. It is a boundary you hit immediately rather than after the fact.

## Charts

Every chart is a thin wrapper. `Chart.yaml` declares a dependency on the upstream chart and `values.yaml` holds the configuration nested under its name. Only what the upstream chart does not provide goes into `templates/`: ClusterIssuer, Ingress, StorageClass, SealedSecret, database CRs.

An application's database is described in that application's own chart and lives in its namespace rather than next to the operator. That keeps backups and database lifecycle tied to a specific service instead of spread across the cluster.

## Networking

MetalLB in L2 mode hands out addresses from a pool on the home network. A single LoadBalancer faces outward — the ingress-nginx service — and everything else is routed through it by hostname. Certificates are issued by cert-manager over DNS-01 through Cloudflare. Internal dashboards such as Longhorn and ArgoCD are additionally restricted to the local network's address range.

## Storage

Two classes. `longhorn` is the default: block storage for databases and small amounts of state. `nfs-media` is RWX over NFS, for bulk data that several pods need to read at once; deleting a PVC there does not erase the underlying data.

## Secrets

Only SealedSecrets are committed to git. They can be decrypted solely by the controller in this cluster, and the private key is never exported. A separate check makes sure a plain Secret never slips into the repository.

## Observability

kube-prometheus-stack for metrics, Loki and Alloy for logs, all of it surfaced in Grafana.

## What is running now

erpnext with MariaDB via the operator, grocy, k8s-pobeda, plus all of the infrastructure listed above. authentik is currently being rebuilt.

## Validation before merge

A merge into `main` goes straight to the live cluster, so changes are checked beforehand: `helm lint`, a render through `helm template`, `kubeconform` against the rendered manifests, and `yamllint` across the repository. The same set of checks is described in `.gitea/workflows/ci.yml`.
