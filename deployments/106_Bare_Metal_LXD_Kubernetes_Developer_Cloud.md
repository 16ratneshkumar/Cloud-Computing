[← Back to Index](../index.md)

# DEPLOYMENT 106 — Bare Metal → LXD + Kubernetes → Developer Cloud

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                Developer Cloud                 │
├──────────────────────────────────────────────────┤
│                LXD + Kubernetes                │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              DEVELOPER WORKLOADS                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Developer K8s namespaces (isolated per team)            │   │
│  │  CI/CD pipelines (Jenkins / GitLab / Tekton)             │   │
│  │  Dev/test environments (ephemeral, fast-delete)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (dev cluster)                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kubeadm or k3s deployed inside LXD containers or VMs   │   │
│  │  Namespace isolation per team                            │   │
│  │  RBAC → team-scoped access                              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s nodes = LXD containers/VMs
┌────────────────────────────▼────────────────────────────────────┐
│              LXD (rapid instance provisioning)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  LXC containers (K8s worker nodes — fast spin-up)        │   │
│  │  LXD VMs (K8s control plane — stronger isolation)        │   │
│  │  ZFS storage (instant clones for new dev environments)   │   │
│  │  OVN networking (per-team VLAN isolation)                │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + KVM (for LXD VMs)                   │
│  ZFS │ KVM │ OVN │ cgroups v2 │ namespaces                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x for VMs) │ RAM │ NVMe (ZFS pool) │ 10GbE NIC        │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Developer platforms, internal K8s as a service, CI/CD infrastructure where speed of new environment creation is critical.

## Pros
- LXD container spin-up in seconds (vs minutes for full VM)
- ZFS instant clone → new K8s node in seconds
- Low overhead per developer namespace
- OVN provides proper network isolation

## Cons
- LXD container K8s nodes share kernel (less isolation than VMs)
- LXD VMs have more overhead
- Not suitable for production multi-tenant