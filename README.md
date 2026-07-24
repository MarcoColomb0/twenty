# twenty

My home Kubernetes cluster, running on bare metal and a VM at home, managed entirely through Git.

[Talos Linux](https://www.talos.dev) for the OS, [Flux](https://fluxcd.io) for GitOps, [Cilium](https://cilium.io)
for networking, [SOPS](https://github.com/getsops/sops) for secrets. Everything that runs in the cluster
is declared in this repository — nothing is applied by hand.

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
| [k8s-gateway](https://github.com/k8s-gateway/k8s_gateway) | Resolves cluster hostnames on the home network (split DNS). |
| [cloudflared](https://github.com/cloudflare/cloudflared) | Tunnel for public traffic — no ports forwarded on the router. |
| [Reloader](https://github.com/stakater/Reloader) | Restarts workloads when their ConfigMaps or Secrets change. |

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

Grafana, Prometheus, and Alertmanager are reachable on the internal gateway only. Kromgo is the one
piece deliberately exposed publicly, because GitHub has to be able to fetch the badges on this page.

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

Cilium runs in native routing mode with `kubeProxyReplacement` enabled, so there is no kube-proxy and
no overlay. Gateway load balancer IPs are handed out by Cilium's LB-IPAM and announced on the LAN over
L2. Public traffic arrives through a Cloudflare tunnel rather than an open port.

Anything attached to the `envoy-internal` gateway resolves only on the home network; anything on
`envoy-external` is reachable from the internet.

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

Secrets are committed encrypted with SOPS and age, and decrypted in-cluster by Flux. Files matching
`*.sops.yaml` are never readable in this repository.

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
