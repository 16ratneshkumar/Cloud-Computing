[← Back to Index](../index.md)

# DEPLOYMENT 30 — Bare Metal → k3s → Containers (Lightweight Kubernetes)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│      Containers (Lightweight Kubernetes)       │
├──────────────────────────────────────────────────┤
│                      k3s                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBECTL / HELM / FLUX                              │
│  (same Kubernetes API — compatible tooling)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │  port 6443
┌────────────────────────────▼────────────────────────────────────┐
│              k3s SERVER (Control Plane — single binary)         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  /usr/local/bin/k3s  (ALL-IN-ONE ~70MB binary)          │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  kube-apiserver (embedded)                          │ │   │
│  │  │  kube-scheduler (embedded)                          │ │   │
│  │  │  kube-controller-manager (embedded)                 │ │   │
│  │  │  SQLite (default) OR etcd / external DB (HA mode)   │ │   │
│  │  │  containerd (embedded CRI)                          │ │   │
│  │  │  Flannel (embedded CNI — simple VXLAN)              │ │   │
│  │  │  Traefik (embedded ingress controller)              │ │   │
│  │  │  CoreDNS (embedded DNS)                             │ │   │
│  │  │  Local-path-provisioner (embedded storage)          │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  k3s agent on worker nodes
┌────────────────────────────▼────────────────────────────────────┐
│              k3s AGENT NODES (Worker Nodes)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  /usr/local/bin/k3s agent                                │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  kubelet (embedded) │ kube-proxy (embedded)        │  │   │
│  │  │  containerd (embedded CRI)                         │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │             PODS                                   │  │   │
│  │  │  ┌──────────────┐  ┌──────────────┐              │  │   │
│  │  │  │  Pod A       │  │  Pod B       │              │  │   │
│  │  │  │  (app)       │  │  (service)   │              │  │   │
│  │  │  └──────────────┘  └──────────────┘              │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (ARM64 or x86)                  │
│  cgroups │ namespaces │ iptables │ overlayfs                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Raspberry Pi / Intel NUC / ARM SBC / Edge Server              │
│  As low as 512MB RAM per node │ SD card / eMMC / SSD storage    │
└─────────────────────────────────────────────────────────────────┘

k3s HA MODE (3 servers + etcd):
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  k3s Server 1│  │  k3s Server 2│  │  k3s Server 3│
│  (etcd node) │◄─┤  (etcd node) ├─►│  (etcd node) │
└──────────────┘  └──────────────┘  └──────────────┘
       ↕                  ↕                  ↕
┌──────────────────────────────────────────────────┐
│  k3s Agent Nodes (worker nodes)                  │
└──────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- k3s bundles everything (apiserver, scheduler, controller-manager, kubelet, containerd, Flannel CNI, CoreDNS, Traefik) into a single ~70MB binary
- Single node: k3s uses SQLite instead of etcd — dramatically reducing resource usage
- HA mode: k3s switches to embedded etcd across 3+ server nodes
- k3s agent runs on worker nodes — connects back to server via TLS

## Use Case
Edge computing, IoT gateways, Raspberry Pi clusters, resource-constrained environments, dev/test, and remote deployments needing minimal footprint Kubernetes.

## Pros
- Single binary — easy install (curl | bash)
- Works on ARM (Raspberry Pi, Jetson, AWS Graviton)
- Only ~512MB RAM for a basic cluster
- Full Kubernetes API compatible
- Fast startup

## Cons
- SQLite backend not for large HA production
- Some enterprise K8s features absent
- Less tested at hyperscale (1000+ nodes)
- Flannel CNI less feature-rich than Calico/Cilium