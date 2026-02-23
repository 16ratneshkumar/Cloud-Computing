[← Back to Index](../index.md)

# DEPLOYMENT 80 — Bare Metal → LXD → VMs (QEMU) → Kubernetes (Cluster API)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│            Kubernetes (Cluster API)            │
├──────────────────────────────────────────────────┤
│                   VMs (QEMU)                   │
├──────────────────────────────────────────────────┤
│                      LXD                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS (tenant clusters)             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  K8s cluster A (prod)        K8s cluster B (staging)    │   │
│  │  Running inside LXD VMs      Running inside LXD VMs     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s nodes = LXD VMs
┌────────────────────────────▼────────────────────────────────────┐
│              LXD VIRTUAL MACHINES (QEMU-backed)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vm: k8s-ctrl-a-1    vm: k8s-wkr-a-1   vm: k8s-ctrl-b-1 │   │
│  │  Ubuntu 22.04         Ubuntu 22.04      Ubuntu 22.04     │   │
│  │  kubeadm / k3s        kubeadm / k3s     kubeadm / k3s   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  LXD CAPL (Cluster API Provider LXD)
┌────────────────────────────▼────────────────────────────────────┐
│              LXD CLUSTER (management cluster)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  LXD daemon (lxd) on each node                           │   │
│  │  dqlite (Raft consensus) — cluster state                 │   │
│  │  REST API (:8443)                                        │   │
│  │  Storage: ZFS (local) or Ceph (distributed)              │   │
│  │  Network: OVN (SDN for VM networking)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + KVM (each LXD node)                 │
│  kvm.ko │ QEMU (for LXD VMs) │ ZFS │ OVN │ cgroups             │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  LXD Node 1 │ LXD Node 2 │ LXD Node 3 (min for HA)            │
│  CPU (VT-x) │ RAM 256GB+ │ NVMe (ZFS pool) │ 25GbE NIC        │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
LXD-centric infrastructure teams wanting Kubernetes clusters managed via Cluster API, using LXD VMs as K8s nodes.