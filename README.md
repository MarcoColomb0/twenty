<div align="center">

# twenty

_A home Kubernetes cluster on Talos Linux, managed entirely through Git._

</div>

<div align="center">

![Talos](https://kromgo.marco.wf/badges/talos_version)
![Kubernetes](https://kromgo.marco.wf/badges/kubernetes_version)
![Flux](https://kromgo.marco.wf/badges/flux_version)
![Age](https://kromgo.marco.wf/badges/cluster_age)
![Uptime](https://kromgo.marco.wf/badges/cluster_uptime)
![Alerts](https://kromgo.marco.wf/badges/cluster_alert_count)

![Nodes](https://kromgo.marco.wf/badges/cluster_node_count)
![Pods](https://kromgo.marco.wf/badges/cluster_pod_count)
![CPU](https://kromgo.marco.wf/badges/cluster_cpu_usage)
![Memory](https://kromgo.marco.wf/badges/cluster_memory_usage)
![Volumes](https://kromgo.marco.wf/badges/longhorn_volume_count)
![Disk Used](https://kromgo.marco.wf/badges/longhorn_capacity_used)

</div>

---

## 🍼 Overview

Three Talos nodes, all control plane, all schedulable. [Flux](https://fluxcd.io) reconciles
everything in this repository into the cluster — including DNS records, TLS certificates and OIDC
clients. Nothing is applied by hand, and anything that cannot be committed in the clear is encrypted
with SOPS and age.

The numbers above are queried from the cluster's own Prometheus and rendered by
[Kromgo](https://github.com/home-operations/kromgo). They are real, and they update on their own.

| CPU | Memory |
|---|---|
| ![CPU usage](https://kromgo.marco.wf/graphs/cluster_cpu?last=24h) | ![Memory usage](https://kromgo.marco.wf/graphs/cluster_memory?last=24h) |

| Running pods | Network throughput |
|---|---|
| ![Running pods](https://kromgo.marco.wf/graphs/cluster_pods?last=24h) | ![Network throughput](https://kromgo.marco.wf/graphs/cluster_network?last=24h) |

## 🔧 Hardware

| Node | Address | Kind | Role |
|---|---|---|---|
| `twenty-lp01` | `192.168.100.201` | LattePanda N150, bare metal | Control plane + workloads |
| `twenty-hp01` | `192.168.100.202` | HP mini PC, i5-8500T | Control plane + workloads |
| `twenty-asus01` | `192.168.100.203` | ASUS PN50, Ryzen | Control plane + workloads |

The API server sits behind a shared VIP at `192.168.100.100`.

Every node is its own physical box. It used to run four, but `twenty-01` and `twenty-virt01` were both
VMs on a single LattePanda — Longhorn spreads replicas across *nodes* and cannot know that, so "two
healthy replicas" could still mean one machine. That stopped being theoretical when the host took two
of the four control plane nodes down at once. The pair was retired and the LattePanda rebuilt as a
single bare-metal node, which is `twenty-lp01`.

Three etcd members means a quorum of two, so one node can be lost without stopping the API, and
Longhorn always has a third node to rebuild a replica onto. Both of those were briefly untrue while
the cluster sat at two nodes, and the two-of-two window is worth avoiding — losing either box took the
API down outright rather than degrading it.

`twenty-hp01` and `twenty-lp01` both have Intel iGPUs and carry the label the Intel GPU device plugin
selects on; Immich transcodes on that. The AMD iGPU in `twenty-asus01` needs a different device plugin
and is not yet exposed.

## 🖥️ Technology Stack

|   | Name | Purpose |
|---|---|---|
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/talos.svg"> | [Talos Linux](https://www.talos.dev) | Immutable, API-driven OS. No SSH, no shell, no package manager. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/kubernetes.svg"> | [Kubernetes](https://kubernetes.io) | Three-node cluster, etcd quorum of two, so one node can be lost. |
| <img width="28" src="https://github.com/cncf/artwork/raw/main/projects/flux/icon/color/flux-icon-color.svg"> | [Flux](https://fluxcd.io) | Reconciles this repository into the cluster. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/cilium.svg"> | [Cilium](https://cilium.io) | CNI in native routing mode, kube-proxy replacement, L2 announcements for load balancer IPs. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/coredns.svg"> | [CoreDNS](https://coredns.io) | Cluster DNS, with the cluster domain forwarded to k8s-gateway. |
| <img width="28" src="https://github.com/cncf/artwork/raw/main/projects/envoy/icon/color/envoy-icon-color.svg"> | [Envoy Gateway](https://gateway.envoyproxy.io) | Gateway API ingress, internal and external. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/cert-manager.svg"> | [cert-manager](https://cert-manager.io) | Wildcard TLS from Let's Encrypt via DNS-01, on the six-day profile. |
| <img width="28" src="https://raw.githubusercontent.com/kubernetes-sigs/external-dns/master/docs/img/external-dns.png"> | [external-dns](https://github.com/kubernetes-sigs/external-dns) | Publishes public DNS records to Cloudflare. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/blocky.svg"> | [Blocky](https://github.com/0xERR0R/blocky) | The LAN's resolver. Splits the cluster domain to k8s-gateway, everything else upstream over DNS-over-TLS, ads blocked on the way. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/cloudflared.svg"> | [cloudflared](https://github.com/cloudflare/cloudflared) | Tunnel for public traffic — no ports forwarded on the router. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/longhorn.svg"> | [Longhorn](https://longhorn.io) | Replicated block storage, two replicas per volume, default StorageClass. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/postgresql.svg"> | [CloudNativePG](https://cloudnative-pg.io) | PostgreSQL operator. An app that owns a database keeps the `Cluster` beside it. |
| <img width="28" src="https://avatars.githubusercontent.com/u/80352373"> | [Dragonfly](https://www.dragonflydb.io) | Redis-compatible in-memory store. |
| <img width="28" src="https://raw.githubusercontent.com/backube/volsync/main/docs/media/volsync.svg"> | [VolSync](https://volsync.readthedocs.io) | Backs volumes up with restic, encrypted before anything leaves the cluster. |
| <img width="28" src="https://avatars.githubusercontent.com/u/99631794"> | [Spegel](https://spegel.dev) | Peer-to-peer image mirror. Nodes pull layers from each other over the LAN, so only the first pull of an image leaves the network. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/prometheus.svg"> | [Prometheus](https://prometheus.io) | Metrics, 5-day retention on Longhorn. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/grafana.svg"> | [Grafana](https://grafana.com) | Dashboards. No login form at all — Pocket ID only. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/alertmanager.svg"> | [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) | Routes alerts to ntfy. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/ntfy.svg"> | [ntfy](https://ntfy.sh) | Push notifications, published so alerts arrive when away from home. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/gatus.svg"> | [Gatus](https://gatus.io) | Uptime checks, and the closest thing here to a service index. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/crowdsec.svg"> | [CrowdSec](https://crowdsec.net) | Reads the external gateway's logs and issues bans, enforced as an `ext_authz` check. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/pocket-id.svg"> | [Pocket ID](https://github.com/pocket-id/pocket-id) | OIDC provider. Passkeys only — there is no password to phish. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/reloader.svg"> | [Reloader](https://github.com/stakater/Reloader) | Restarts workloads when their ConfigMaps or Secrets change. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/kubernetes-dashboard.svg"> | [Intel device plugins](https://github.com/intel/intel-device-plugins-for-kubernetes) | Advertises the Intel iGPUs on `twenty-hp01` and `twenty-lp01` as `gpu.intel.com/i915`. A `hostPath` mount is not enough: it shows the render node to the container but never grants the device cgroup entry. |

## 📦 Applications

|   | Name | Notes |
|---|---|---|
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/immich.svg"> | [Immich](https://immich.app) | Photos and video. Internal gateway only, Pocket ID with no password login at all, transcodes on the iGPU. Settings live in its database rather than a mounted file, because `IMMICH_CONFIG_FILE` makes the admin UI read-only. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/immich.svg"> | [immich-public-proxy](https://github.com/alangrainger/immich-public-proxy) | The one publicly exposed half of Immich. Serves shared album links and nothing else, so the library stays off the internet. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/affine.svg"> | [AFFiNE](https://affine.pro) | Notes and whiteboards. Its schema migration runs as an init container, which is how upstream orders it too. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/karakeep.svg"> | [Karakeep](https://karakeep.app) | Bookmarks and read-later. Crawler and search index run as sidecars reached over localhost, so neither is on the cluster network. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/actual-budget.svg"> | [Actual Budget](https://actualbudget.org) | Budgeting, SQLite-backed. Pocket ID is enforced — there is no server password to fall back to. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/ghostfolio.svg"> | [Ghostfolio](https://ghostfol.io) | Portfolio tracker, backed by CloudNativePG and Dragonfly. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/it-tools.svg"> | [IT-Tools](https://github.com/CorentinTh/it-tools) | Offline developer utilities — encoders, converters, generators. |
| <img width="28" src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/kubernetes.svg"> | echo | Trivial HTTP echo service, used to verify ingress and DNS end to end. |

## 🔑 Identity

| Component | Purpose |
|---|---|
| [Pocket ID](https://github.com/pocket-id/pocket-id) | OIDC provider. Passkeys only — there is no password to phish. Backed by CloudNativePG. |
| [pocket-id-operator](https://github.com/aclerici38/pocket-id-operator) | Manages the OIDC clients, users and groups inside Pocket ID as Kubernetes resources, so they live in this repository rather than in a web UI. |

Adding SSO to an app is a `PocketIDOIDCClient` next to it. The operator creates the client, writes the
credentials to a `Secret`, and the app reads them — nothing is clicked, and deleting the resource
rebuilds it identically. Grafana, Immich, Karakeep, AFFiNE and Gatus all sign in this way.

Grafana has no login form at all: Pocket ID is the only way in through a browser, so there is no
Grafana-local password to phish or reuse. The admin account still answers on the HTTP API over basic
auth, which is the way back in if Pocket ID is unreachable.

## 🛡️ Security

| Component | Purpose |
|---|---|
| [CrowdSec](https://crowdsec.net) | Reads the external gateway's access logs, scores behaviour against community scenarios, and issues bans. Enrolled in the CrowdSec console as `twenty`. |
| [envoy-proxy-bouncer](https://github.com/kdwils/envoy-proxy-crowdsec-bouncer) | Enforces those bans, as an `ext_authz` check on every request the external gateway serves. |

The agent runs as a DaemonSet and only tails `envoy-external` — the internal gateway serves the LAN,
which is whitelisted anyway. That is also why nothing app-specific is acquired: a service published
through the external gateway is already parsed there, with the real client resolved from
`X-Forwarded-For`, while a LAN-only service would only ever contribute private addresses. Immich is
the worked example — the hub has a `gauth-fr/immich` collection, but Immich answers on the internal
gateway and its share links arrive via a proxy, so its own logs would show either a whitelisted LAN
address or the proxy's pod IP. `immich-public-proxy` needs no collection of its own for the same
reason: it sits behind `envoy-external` and is parsed there already.

AppSec runs alongside it as an in-band WAF, combining targeted virtual patching with the generic
rules and the OWASP core rule set.

Two things keep this from locking the house from the inside. `crowdsecurity/whitelists` drops RFC1918
in the parsing pipeline, and because split DNS resolves the cluster domain straight to the
`envoy-external` VIP on the LAN, local traffic really does arrive with a private source address rather
than a Cloudflare one. AppSec is evaluated in the bouncer instead of the parsers, so it needs its own
guard: the bouncer exempts the same private ranges. The `SecurityPolicy` also sets `failOpen`, so a
bouncer that is down costs remediation and not the gateway.

## 📊 Observability

Every component that exports metrics is scraped: Cilium, Envoy, Longhorn, CloudNativePG, Flux,
cert-manager, Blocky, Pocket ID, and CrowdSec — the agent on every node, plus AppSec, the LAPI and
the bouncer. Grafana carries the kubernetes-mixin dashboards plus one per app,
pinned by revision or release tag so upstream cannot quietly change what a panel shows.

Prometheus and Alertmanager are reachable on the internal gateway only. Neither has any
authentication, so publishing them would hand over every metric and the ability to silence alerts.

Grafana is published, because it is the one of the three that can defend itself: it has no login form
at all, so Pocket ID is the only way in, and the external gateway puts it behind the CrowdSec bouncer
and AppSec like everything else out there. Kromgo is public too, because GitHub has to be able to
fetch the badges on this page.

## 💾 Backups

Backups run to a [rustfs](https://rustfs.com) S3 endpoint on a Raspberry Pi 4 on the LAN.

**restic encrypts client-side.** The mover pods chunk, deduplicate, compress and encrypt inside the
cluster, so the Pi only ever receives finished blobs and does none of the work — which is what makes
a 4GB Pi an adequate destination. It also means the Pi holds nothing readable: server-side encryption
on a box you own stores the key next to the data and protects against nothing if the box is stolen.

Adding a volume to the backup set is a component reference and three lines of substitution — no new
secret, because the credentials live once in `cluster-secrets`:

```yaml
# kubernetes/apps/<ns>/<app>/app/kustomization.yaml
components:
  - ../../../../components/volsync
```
```yaml
# kubernetes/apps/<ns>/<app>/ks.yaml
postBuild:
  substitute:
    APP: myapp
    VOLSYNC_CLAIM: myapp-data
```

Each app gets its own restic repository under `${APP}`. They cannot share one: VolSync hardcodes the
restic host to `volsync` for every mover, so a single repository would apply one retention policy
across everything in it and quietly drop other apps' snapshots.

Retention is 7 daily, 4 weekly, 6 monthly. `forget` runs every cycle and is cheap; `prune` actually
repacks the repository and runs fortnightly.

Two deliberate choices worth knowing:

- **`copyMethod: Direct`**, not `Snapshot`. On Longhorn a snapshot is restored into a whole new volume
  sized by the source claim's *request*, so Immich would provision another 150Gi and copy 56GB into it
  nightly — and with over-provisioning capped at 100%, no node could schedule it at any replica count.
- **Scratch volumes use `longhorn-single`.** The clone and the restic cache are ephemeral; the durable
  copy is the repository, so replicating them protects nothing.

### What is and is not covered

| Data | Covered by | Status |
|---|---|---|
| `immich-data`, `affine-storage`, `karakeep-data`, `actual-data` | VolSync + restic | ✅ |
| PostgreSQL (Immich, AFFiNE, Ghostfolio, Pocket ID) | barman-cloud → `twenty-postgres` | ⏳ not yet deployed |
| Prometheus, Meilisearch index, CrowdSec, ntfy cache | nothing, on purpose | expires or rebuilds itself |

> **The databases are not backed up yet.** Until barman-cloud lands, a CloudNativePG cluster survives
> on its replicas alone — losing both instances loses the data.

## 🚑 Restore

### The one thing to protect

Everything below assumes you still have the **age private key**. rustfs holds nothing but ciphertext,
and the restic passphrase lives in `cluster-secrets` encrypted with that key. No age key means no
passphrase, which means the backups are permanently unreadable. Keep a copy off the cluster — on
paper or a USB stick, not only on the machine it protects.

### Restore one volume

VolSync restores through a `ReplicationDestination`, which writes the repository's latest snapshot
into a new claim. Nothing overwrites the live volume until you swap it in.

```sh
# 1. Stop the app so nothing writes while the volume is swapped
kubectl scale deploy/<app> -n <ns> --replicas=0

# 2. Pull the latest snapshot into a new claim
kubectl apply -n <ns> -f - <<'EOF'
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: <app>-restore
spec:
  trigger:
    manual: restore-once
  restic:
    repository: <app>-volsync-restic     # the Secret the component already created
    copyMethod: Direct
    destinationPVC: <app>-data           # an EXISTING empty claim, or drop this line
    capacity: 10Gi                       #   and let VolSync provision one
    accessModes: ["ReadWriteOnce"]
    storageClassName: longhorn
    cacheStorageClassName: longhorn-single
    cacheCapacity: 8Gi
    moverSecurityContext: { runAsUser: 1000, runAsGroup: 1000, fsGroup: 1000 }
EOF

# 3. Watch it land
kubectl get replicationdestination -n <ns> <app>-restore -w

# 4. Start the app again
kubectl scale deploy/<app> -n <ns> --replicas=1
```

To restore an *older* snapshot rather than the latest, add `previous: N` under `spec.restic` — `N` is
how many snapshots back to step.

To inspect a repository by hand, run restic anywhere with the same four environment variables the
component builds (`RESTIC_REPOSITORY`, `RESTIC_PASSWORD`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`):

```sh
restic snapshots
restic ls latest
restic restore latest --target /tmp/out
```

### Restore one database

CloudNativePG rebuilds a replica by itself. If one instance is broken but the primary is healthy,
delete the replica's claim and pod and it re-clones from the primary:

```sh
kubectl delete pvc  -n <ns> postgres-<app>-2 --wait=false
kubectl delete pod  -n <ns> postgres-<app>-2
```

For a logical copy before touching anything — worth doing while barman-cloud is still pending:

```sh
kubectl exec -n <ns> postgres-<app>-1 -c postgres -- \
  pg_dump -U postgres -d <app> --no-owner --no-acl -Fc > <app>.dump
```

Restoring that dump into a fresh cluster leaves every object owned by `postgres`, which the app
cannot read — reassign ownership afterwards or it fails with `permission denied for table ...`.

### Rebuild the whole cluster

Ordering matters more than any individual step.

1. **Recover the age key.** Without it nothing else in this list is possible.
2. **Reinstall Talos.** `just talos generate-config`, then `just talos apply-node <ip>` per node.
   Node configuration is entirely in [`talos/`](./talos) — nothing was set by hand.
3. **Bootstrap Flux** from [`bootstrap/`](./bootstrap). It will reconcile this repository and rebuild
   every namespace, workload, certificate, DNS record and OIDC client on its own.
4. **Wait for storage.** Longhorn, the snapshot controller and VolSync must be healthy before any
   restore will work. `kubectl get storageclass` should list `longhorn` and `longhorn-single`.
5. **Restore the databases**, before the apps that use them come up healthy. Until barman-cloud is
   deployed this means restoring `pg_dump` output by hand.
6. **Restore the volumes** with a `ReplicationDestination` per app, as above. The repositories are
   independent, so these can run in parallel — though on one Pi they will queue on disk anyway.
7. **Re-enrol what lives outside git**: CrowdSec's console enrolment and any Pocket ID passkeys
   registered against the old instance.

What a rebuild does *not* need: DNS records, TLS certificates, OIDC clients and database roles are all
declared here and recreated by Flux. What it does need and cannot get from git: the age key, and the
data in the restic repositories.

## 🌐 Networking

| | |
|---|---|
| Pod CIDR | `10.42.0.0/16` |
| Service CIDR | `10.43.0.0/16` |
| API server VIP | `192.168.100.100` |
| Internal gateway | `192.168.100.101` |
| External gateway | `192.168.100.102` |
| k8s-gateway (cluster DNS) | `192.168.100.103` |
| Blocky (LAN DNS) | `192.168.100.104` |

Cilium runs in native routing mode with `kubeProxyReplacement` enabled, so there is no kube-proxy and
no overlay. Load balancer IPs are handed out by Cilium's LB-IPAM and announced on the LAN over L2.
Public traffic arrives through a Cloudflare tunnel rather than an open port.

Anything attached to the `envoy-internal` gateway resolves only on the home network; anything on
`envoy-external` is reachable from the internet.

### DNS

The LAN points at Blocky, which splits queries three ways:

```
LAN client ──► 192.168.100.104  blocky ×3, one per node
                    │
                    ├── cluster domain ──► 192.168.100.103  k8s-gateway
                    │                          └─► envoy-internal .101 / envoy-external .102
                    │
                    └── everything else ──► DNS-over-TLS upstream, ads blocked
```

Nothing here maintains a DNS record by hand. k8s-gateway derives them from the `HTTPRoute` objects in
this repository, so an app becomes resolvable on the LAN the moment its route exists — and
external-dns does the same for public names in Cloudflare.

## 🗂️ Repository layout

```
bootstrap/     Helmfile + scripts to get from bare Talos to a Flux-managed cluster
kubernetes/
  apps/        One directory per namespace, one Flux Kustomization per app
  components/  Reusable Kustomize components (SOPS cluster secrets)
  flux/        The root Kustomization that adopts everything under apps/
talos/         talconfig.yaml and machine patches, rendered by talhelper
```

Each app follows the same shape: a `ks.yaml` declaring the Flux `Kustomization`, and an `app/`
directory holding the `HelmRelease` or plain manifests plus its chart source.

```
kubernetes/apps/<namespace>/<app>/
├── ks.yaml
└── app/
    ├── kustomization.yaml
    ├── ocirepository.yaml
    └── helmrelease.yaml
```

Namespaces map to directories under `kubernetes/apps/`:
`actual`, `affine`, `cert-manager`, `database`, `default`, `flux-system`, `ghostfolio`, `immich`, `karakeep`, `kube-system`, `network`, `observability`, `security`, `storage`.

An app that owns a database gets its own namespace and keeps the `Cluster` beside it. CloudNativePG
publishes the `-app` Secret next to the `Cluster`, and Secrets do not cross namespaces, so splitting
the two means copying a generated credential by hand and watching it drift.

Secrets are committed encrypted with SOPS and age, and decrypted in-cluster by Flux. Files matching
`*.sops.yaml` are never readable in this repository — **this repository is public**, so every value
that should stay private lives inside one of them.

## 🤖 Operating it

The toolchain is pinned in [`.mise.toml`](./.mise.toml); `mise install` gets everything.

```sh
# Reconcile the cluster against this repository right now
just kube reconcile

# Regenerate Talos config after editing talconfig.yaml or a patch
just talos generate-config
just talos apply-node 192.168.100.202

# Upgrade a node, then the cluster
just talos upgrade-node 192.168.100.202
just talos upgrade-k8s
```

Dependencies are kept current by [Renovate](https://www.mend.io/renovate), and every pull request runs
[flux-local](https://github.com/allenporter/flux-local) to diff the rendered manifests before anything
reaches the cluster.

## 🤝 Acknowledgments

Built from [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) and shaped by the
[Home Operations](https://discord.gg/home-operations) community. README structure owes a lot to
[xunholy/k8s-gitops](https://github.com/xunholy/k8s-gitops); icons come from
[Dashboard Icons](https://dashboardicons.com).
