# twenty

My home Kubernetes cluster, running on bare metal and a VM at home, managed entirely through Git.

[Talos Linux](https://www.talos.dev) for the OS, [Flux](https://fluxcd.io) for GitOps, [Cilium](https://cilium.io)
for networking, [Pocket ID](https://github.com/pocket-id/pocket-id) for single sign-on,
[SOPS](https://github.com/getsops/sops) for secrets. Everything that runs in the cluster is declared in
this repository — nothing is applied by hand, including DNS records and OIDC clients.

## Live status

Numbers below are queried from the cluster's own Prometheus and rendered by
[Kromgo](https://github.com/home-operations/kromgo). They are real, and they update on their own.

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
![Disk Total](https://kromgo.marco.wf/badges/longhorn_capacity_total)

![Network In](https://kromgo.marco.wf/badges/cluster_network_rx)
![Network Out](https://kromgo.marco.wf/badges/cluster_network_tx)
![Requests](https://kromgo.marco.wf/badges/envoy_request_rate)

### Last 24 hours

| CPU | Memory |
|---|---|
| ![CPU usage](https://kromgo.marco.wf/graphs/cluster_cpu?last=24h) | ![Memory usage](https://kromgo.marco.wf/graphs/cluster_memory?last=24h) |

| Running pods | Network throughput |
|---|---|
| ![Running pods](https://kromgo.marco.wf/graphs/cluster_pods?last=24h) | ![Network throughput](https://kromgo.marco.wf/graphs/cluster_network?last=24h) |

Prometheus itself is not exposed — Kromgo publishes only the queries defined in
[`kubernetes/apps/observability/kromgo/app/helmrelease.yaml`](./kubernetes/apps/observability/kromgo/app/helmrelease.yaml).

## Hardware

Three nodes, all control plane, all schedulable. Quorum of three, no separate workers.

| Node | Address | Kind | Role |
|---|---|---|---|
| `twenty-01` | `192.168.100.200` | Bare metal | Control plane + workloads |
| `twenty-virt01` | `192.168.100.201` | VM | Control plane + workloads |
| `twenty-hp01` | `192.168.100.202` | HP bare metal | Control plane + workloads |

The API server sits behind a shared VIP at `192.168.100.100`, so losing any single node does not take
`kubectl` with it. Node configuration lives in [`talos/talconfig.yaml`](./talos/talconfig.yaml) and the
patches under [`talos/patches/`](./talos/patches).

## What runs here

### Core

| Component | Purpose |
|---|---|
| [Talos Linux](https://www.talos.dev) | Immutable, API-driven OS. No SSH, no shell, no package manager. |
| [Flux Operator](https://fluxcd.control-plane.io/operator/) + Flux | Reconciles this repository into the cluster. |
| [Cilium](https://cilium.io) | CNI in native routing mode, kube-proxy replacement, L2 announcements for load balancer IPs. |
| [CoreDNS](https://coredns.io) | Cluster DNS. |
| [Envoy Gateway](https://gateway.envoyproxy.io) | Gateway API ingress, internal and external. |
| [cert-manager](https://cert-manager.io) | Wildcard TLS from Let's Encrypt via DNS-01. |
| [external-dns](https://github.com/kubernetes-sigs/external-dns) | Publishes public DNS records to Cloudflare. |
| [k8s-gateway](https://github.com/k8s-gateway/k8s_gateway) | Answers for cluster hostnames, with records derived from `HTTPRoute` and `Service`. |
| [Blocky](https://github.com/0xERR0R/blocky) | The LAN's resolver. Forwards the cluster domain to k8s-gateway, everything else upstream over DNS-over-TLS, and blocks ads on the way. |
| [cloudflared](https://github.com/cloudflare/cloudflared) | Tunnel for public traffic — no ports forwarded on the router. |
| [Reloader](https://github.com/stakater/Reloader) | Restarts workloads when their ConfigMaps or Secrets change. |

### Identity

| Component | Purpose |
|---|---|
| [Pocket ID](https://github.com/pocket-id/pocket-id) | OIDC provider. Passkeys only — there is no password to phish. Backed by CloudNativePG. |
| [pocket-id-operator](https://github.com/aclerici38/pocket-id-operator) | Manages the OIDC clients, users and groups inside Pocket ID as Kubernetes resources, so they live in this repository rather than in a web UI. |

Adding SSO to an app is a `PocketIDOIDCClient` next to it. The operator creates the client, writes the
credentials to a `Secret`, and the app reads them — nothing is clicked, and deleting the resource
rebuilds it identically. Grafana signs in this way.

### Security

| Component | Purpose |
|---|---|
| [CrowdSec](https://crowdsec.net) | Reads the external gateway's access logs, scores behaviour against community scenarios, and issues bans. Enrolled in the CrowdSec console as `twenty`. |
| [envoy-proxy-bouncer](https://github.com/kdwils/envoy-proxy-crowdsec-bouncer) | Enforces those bans, as an `ext_authz` check on every request the external gateway serves. |

The agent runs as a DaemonSet and only tails `envoy-external` — the internal gateway serves the LAN,
which is whitelisted anyway. AppSec runs alongside it as an in-band WAF, combining targeted virtual
patching with the generic rules and the OWASP core rule set.

Two things keep this from locking the house from the inside. `crowdsecurity/whitelists` drops RFC1918
in the parsing pipeline, and because split DNS resolves the cluster domain straight to the
`envoy-external` VIP on the LAN, local traffic really does arrive with a private source address rather
than a Cloudflare one. AppSec is evaluated in the bouncer instead of the parsers, so it needs its own
guard: the bouncer exempts the same private ranges. The `SecurityPolicy` also sets `failOpen`, so a
bouncer that is down costs remediation and not the gateway.

### Storage and data

| Component | Purpose |
|---|---|
| [Longhorn](https://longhorn.io) | Replicated block storage, two replicas per volume, default StorageClass. |
| [CloudNativePG](https://cloudnative-pg.io) | PostgreSQL operator. |
| [Dragonfly](https://www.dragonflydb.io) | Redis-compatible in-memory store. |

### Observability

| Component | Purpose |
|---|---|
| [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts) | Prometheus (14d retention on Longhorn), Alertmanager, Grafana, kube-state-metrics, node-exporter. |
| [Kromgo](https://github.com/home-operations/kromgo) | Turns whitelisted PromQL into the public badges and graphs above. |

Every component that exports metrics is scraped: Cilium, Envoy, Longhorn, CloudNativePG, Flux,
cert-manager, Blocky and Pocket ID. Grafana carries the kubernetes-mixin dashboards plus one per app,
pinned by revision or release tag so upstream cannot quietly change what a panel shows.

Grafana, Prometheus, and Alertmanager are reachable on the internal gateway only, and Grafana signs in
through Pocket ID. Kromgo is the one piece deliberately exposed publicly, because GitHub has to be
able to fetch the badges on this page.

### Applications

| App | Purpose |
|---|---|
| [Ghostfolio](https://ghostfol.io) | Portfolio tracker, backed by CloudNativePG and Dragonfly. |
| echo | Trivial HTTP echo service, used to verify ingress and DNS end to end. |

## Networking

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

## Repository layout

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

Namespaces map to directories under `kubernetes/apps/`: `cert-manager`, `database`, `default`,
`flux-system`, `kube-system`, `network`, `observability`, `security`, `storage`.

Secrets are committed encrypted with SOPS and age, and decrypted in-cluster by Flux. Files matching
`*.sops.yaml` are never readable in this repository — **this repository is public**, so every value
that should stay private lives inside one of them.

## Operating it

The toolchain is pinned in [`.mise.toml`](./.mise.toml); `mise install` gets everything.

```sh
# Reconcile the cluster against this repository right now
just kube reconcile

# Regenerate Talos config after editing talconfig.yaml or a patch
just talos generate-config
just talos apply-node 192.168.100.200

# Upgrade a node, then the cluster
just talos upgrade-node 192.168.100.200
just talos upgrade-k8s
```

Dependencies are kept current by [Renovate](https://www.mend.io/renovate), and every pull request runs
[flux-local](https://github.com/allenporter/flux-local) to diff the rendered manifests before anything
reaches the cluster.

## Credits

Built from [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) and shaped by the
[Home Operations](https://discord.gg/home-operations) community.
