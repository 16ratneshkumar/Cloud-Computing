[← Back to Index](../index.md)

# DEPLOYMENT 36 — Bare Metal → LXD Cluster → Kubernetes Nodes (as containers)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│        Kubernetes Nodes (as containers)        │
├──────────────────────────────────────────────────┤
│                  LXD Cluster                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes Cluster (running inside LXD containers)      │   │
│  │  Pods → containers inside LXD container → K8s node       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s nodes are LXD containers
┌────────────────────────────▼────────────────────────────────────┐
│              LXD CONTAINERS (acting as K8s nodes)               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  lxd-k8s-ctrl-1   lxd-k8s-ctrl-2   lxd-k8s-ctrl-3      │   │
│  │  (K8s ctrl plane) (K8s ctrl plane) (K8s ctrl plane)     │   │
│  │  Ubuntu 22.04     Ubuntu 22.04     Ubuntu 22.04         │   │
│  │  own IP (LXD net) own IP           own IP               │   │
│  │                                                          │   │
│  │  lxd-k8s-wkr-1    lxd-k8s-wkr-2   lxd-k8s-wkr-3       │   │
│  │  (K8s worker)     (K8s worker)    (K8s worker)          │   │
│  │  Ubuntu 22.04     Ubuntu 22.04    Ubuntu 22.04          │   │
│  │  containerd       containerd      containerd            │   │
│  │  [nested containers inside LXD container]               │   │
│  └──────────────────────────────────────────────────────────┘   │
│  Needs: security.nesting=true on LXD containers                 │
│         security.privileged=true (or user NS remapping)         │
└────────────────────────────┬────────────────────────────────────┘
                             │  LXD cluster manages container placement
┌────────────────────────────▼────────────────────────────────────┐
│              LXD CLUSTER (across bare metal nodes)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  LXD Node 1 (BM)    LXD Node 2 (BM)    LXD Node 3 (BM)  │   │
│  │  dqlite (Raft)   ←→  dqlite (follower) ←→  dqlite       │   │
│  │  lxd daemon          lxd daemon             lxd daemon   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  Shared storage: ZFS / Ceph / NFS (for container migration)     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (each BM node)                  │
│  cgroups v2 │ namespaces │ overlayfs │ nested namespace support  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Node 1              Node 2              Node 3                │
│  CPU│RAM│NVMe        CPU│RAM│NVMe        CPU│RAM│NVMe           │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- LXD containers are configured with `security.nesting=true` to allow running nested containers (containerd inside LXD container)
- Each LXD container runs a full Ubuntu system acting as a K8s node
- K8s control plane and workers are standard kubeadm clusters — they just happen to run inside LXD containers
- LXD cluster manages container placement across physical nodes

## Use Case
High-density Kubernetes lab environments, multi-tenant dev clouds, training environments needing many K8s nodes on few physical machines.

## Pros
- Very high density — many K8s nodes per physical host
- Fast LXD container provisioning (seconds)
- Easy to scale up/down K8s nodes
- Good for labs and CI test clusters

## Cons
- Nested namespaces — complex cgroup and security config
- Not recommended for production multi-tenant (shared kernel)
- Kernel version constraints
- Some K8s features behave differently in nested environment