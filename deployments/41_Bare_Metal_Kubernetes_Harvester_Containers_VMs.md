[← Back to Index](../index.md)

# DEPLOYMENT 41 — Bare Metal → Kubernetes → Harvester → Containers + VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                Containers + VMs                │
├──────────────────────────────────────────────────┤
│                   Harvester                    │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT PLANE                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Harvester UI (web dashboard — port 443)                 │   │
│  │  Rancher (multi-cluster management, optional)            │   │
│  │  kubectl / Terraform Harvester provider                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              HARVESTER HCI PLATFORM                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CONTAINER WORKLOADS        VM WORKLOADS                 │   │
│  │  ┌──────────────────┐       ┌──────────────────────────┐ │   │
│  │  │  K8s Pods         │       │  KubeVirt VMs            │ │   │
│  │  │  (system +        │       │  ┌──────────────────┐    │ │   │
│  │  │   user workloads) │       │  │  Windows Server  │    │ │   │
│  │  └──────────────────┘       │  │  Ubuntu Linux    │    │ │   │
│  │                             │  └──────────────────┘    │ │   │
│  │                             └──────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  HARVESTER CORE COMPONENTS (all as K8s controllers)      │   │
│  │  ┌─────────────────┐  ┌────────────────────────────────┐ │   │
│  │  │  KubeVirt        │  │  Longhorn (distributed storage)│ │   │
│  │  │  (VM mgmt)       │  │  → block storage for VMs      │ │   │
│  │  └─────────────────┘  └────────────────────────────────┘ │   │
│  │  ┌─────────────────┐  ┌────────────────────────────────┐ │   │
│  │  │  Multus CNI      │  │  VLAN-aware networking         │ │   │
│  │  │  (multi-NIC      │  │  (VM network isolation)        │ │   │
│  │  │   for VMs)       │  │                                │ │   │
│  │  └─────────────────┘  └────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RKE2 (Rancher Kubernetes Engine 2) — K8s distribution  │   │
│  │  kube-apiserver │ etcd │ scheduler │ controllers         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL + LONGHORN DISK                 │
│  kvm.ko │ containerd │ Longhorn agents on each node             │
│  Longhorn replicates VM disk data across nodes                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Node 1 (master+worker)  Node 2 (worker)  Node 3 (worker)      │
│  CPU│RAM│NVMe (Longhorn)  CPU│RAM│NVMe     CPU│RAM│NVMe         │
│  Minimum 3 nodes for production, 1 node for dev                 │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Harvester installs as an OS (based on openSUSE Leap), bootstrapping RKE2 (K8s) automatically
- KubeVirt provides VM management within K8s
- Longhorn provides distributed block storage — VM disks are Longhorn volumes replicated across nodes
- Multus CNI allows VMs to have multiple network interfaces (e.g., management + data VLANs)
- Rancher can manage multiple Harvester clusters from a central dashboard

## Use Case
Open-source VMware alternative for organizations wanting cloud-native HCI. Used to migrate VMware workloads while gaining Kubernetes-native operations.

## Pros
- Full open-source HCI (no licensing)
- Kubernetes-native from the ground up
- Rancher integration for multi-site management
- Built-in distributed storage (Longhorn)
- Active SUSE/Rancher backing

## Cons
- Longhorn storage can be limiting at very high throughput
- Still maturing — 1.x releases
- Requires Kubernetes expertise
- Less mature Windows VM support vs VMware