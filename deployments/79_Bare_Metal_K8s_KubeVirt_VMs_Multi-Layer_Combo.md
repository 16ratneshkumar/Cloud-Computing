[← Back to Index](../index.md)

# DEPLOYMENT 79 — Bare Metal → K8s → KubeVirt → VMs (Multi-Layer Combo)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│            VMs (Multi-Layer Combo)             │
├──────────────────────────────────────────────────┤
│                    KubeVirt                    │
├──────────────────────────────────────────────────┤
│                      K8s                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              COMPLEX WORKLOAD LAYER                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container pods (cloud-native apps)                      │   │
│  │  KubeVirt VMs (legacy apps)                              │   │
│  │  Kata container pods (secure multi-tenant containers)    │   │
│  │  → all managed by kubectl simultaneously                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (unified control plane)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RuntimeClass: runc          → regular containers        │   │
│  │  RuntimeClass: kata-qemu     → Kata MicroVM containers   │   │
│  │  VirtualMachine CRD          → KubeVirt full VMs         │   │
│  │                                                          │   │
│  │  All workload types:                                     │   │
│  │  → same kubectl API                                      │   │
│  │  → same networking (CNI / Multus)                        │   │
│  │  → same storage (CSI / PVC)                              │   │
│  │  → same RBAC and security policies                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Storage: Rook-Ceph CSI (RBD for VMs, CephFS for pods)  │   │
│  │  Network: Multus + OVN-K8s (VLANs for VMs)              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + containerd + LINUX KERNEL                    │
│  kvm.ko │ containerd (runc + kata shims) │ QEMU per KubeVirt VM │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x) │ RAM 512GB+ │ NVMe + HDDs │ 25GbE NIC             │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Enterprise consolidation: running containers, secure containers, and full VMs from one Kubernetes control plane. Migration path from VMware while onboarding new cloud-native apps simultaneously.