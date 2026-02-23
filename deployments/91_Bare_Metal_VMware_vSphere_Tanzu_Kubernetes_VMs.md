[← Back to Index](../index.md)

# DEPLOYMENT 91 — Bare Metal → VMware vSphere + Tanzu → Kubernetes + VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                Kubernetes + VMs                │
├──────────────────────────────────────────────────┤
│             VMware vSphere + Tanzu             │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS (VMware unified platform)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Traditional VMs (Windows, Linux — vSphere managed)      │   │
│  │  Kubernetes clusters (Tanzu managed K8s)                 │   │
│  │  Containers (Tanzu Application Platform — Cloud Foundry) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              VMWARE TANZU + vSPHERE MANAGEMENT                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vCenter Server (VM + Tanzu cluster management)          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Tanzu Kubernetes Grid (TKG)                       │  │   │
│  │  │  → provisions K8s clusters on vSphere VMs          │  │   │
│  │  │  vSphere with Tanzu (Supervisor)                   │  │   │
│  │  │  → K8s API directly on vSphere (no separate TKG)  │  │   │
│  │  │  Tanzu Application Platform (TAP)                  │  │   │
│  │  │  → developer self-service, supply chain security   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  vSphere features: DRS │ HA │ vMotion │ vSAN │ NSX-T     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              ESXI HOSTS (hypervisor nodes)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Guest VMs (Windows/Linux workloads)                     │   │
│  │  Tanzu K8s cluster node VMs (Ubuntu-based)               │   │
│  │  VMkernel (ESXi hypervisor kernel)                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vSAN: local NVMe pooled across ESXi hosts               │   │
│  │  NSX-T: software-defined networking (micro-segmentation) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  VMware HCL-certified hardware                                  │
│  Intel Xeon / AMD EPYC │ RAM 512GB+ │ NVMe (vSAN) │ 25GbE NIC │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Large enterprises standardized on VMware wanting to modernize to Kubernetes while retaining existing VMware investment. Bridges traditional VM world to cloud-native.

## Pros
- Unified VMware management for VMs + K8s
- Mature HA, DRS, vMotion for all workloads
- NSX-T micro-segmentation for both VMs and K8s
- vSAN for integrated storage

## Cons
- Dramatically increased cost post-Broadcom 2024 acquisition
- Very complex licensing tiers
- Driving customers away toward Nutanix, Proxmox, OpenShift
- Deep VMware lock-in