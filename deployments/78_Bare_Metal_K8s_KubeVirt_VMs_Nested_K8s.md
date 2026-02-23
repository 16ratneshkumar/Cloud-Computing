[← Back to Index](../index.md)

# DEPLOYMENT 78 — Bare Metal → K8s → KubeVirt VMs → Nested K8s

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Nested K8s                   │
├──────────────────────────────────────────────────┤
│                  KubeVirt VMs                  │
├──────────────────────────────────────────────────┤
│                      K8s                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              INNER KUBERNETES (L2 — inside VMs)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Inner K8s cluster (tenant-controlled)                   │   │
│  │  Pods, services, ingress → all inside VM-based nodes     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  each inner K8s node = a KubeVirt VM
┌────────────────────────────▼────────────────────────────────────┐
│              KUBEVIRT VMs (inner K8s nodes)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VM: inner-ctrl-1   VM: inner-wkr-1   VM: inner-wkr-2   │   │
│  │  Ubuntu 22.04        Ubuntu 22.04      Ubuntu 22.04      │   │
│  │  kubeadm-joined     kubeadm-joined    kubeadm-joined     │   │
│  │  containerd         containerd        containerd         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  KubeVirt manages VMs in outer K8s
┌────────────────────────────▼────────────────────────────────────┐
│              OUTER KUBERNETES (L1 — management cluster)         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KubeVirt operator (manages VMs)                         │   │
│  │  Cluster API (CAPI) + CAPK (KubeVirt provider)           │   │
│  │  → CAPI provisions inner K8s clusters via KubeVirt VMs   │   │
│  │  CDI (disk image import for inner cluster node images)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (outer cluster nodes)           │
│  kvm.ko │ nested KVM (if inner VMs need nested virt)           │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x) │ RAM 512GB+ │ NVMe (Ceph/Longhorn) │ 25GbE NIC   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Kubernetes-as-a-Service on Kubernetes — hyperscalers offering tenant K8s clusters. CAPK (Cluster API Provider KubeVirt) is the key enabler.

## Pros
- K8s-native multi-cluster provisioning
- Strong VM isolation between tenant clusters
- CAPI provides declarative cluster lifecycle
- Active development by KubeVirt community

## Cons
- VM overhead for K8s nodes
- Complex CAPI + KubeVirt expertise required
- Not yet mainstream production