[← Back to Index](../index.md)

# DEPLOYMENT 92 — Bare Metal → Azure Stack HCI → Windows/Linux VMs + AKS

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│            Windows/Linux VMs + AKS             │
├──────────────────────────────────────────────────┤
│                Azure Stack HCI                 │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Windows Server VMs │ Linux VMs │ AKS Arc K8s clusters   │   │
│  │  Azure Arc-enabled data services (Arc SQL, Arc PG)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              AZURE STACK HCI MANAGEMENT                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Azure Portal (manages on-premises HCI via Azure Arc)    │   │
│  │  Windows Admin Center (local web UI)                     │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  AKS on Azure Stack HCI (K8s clusters)             │  │   │
│  │  │  Azure Arc (extends Azure mgmt to on-premises)      │  │   │
│  │  │  Azure Monitor (telemetry to Azure cloud)           │  │   │
│  │  │  Microsoft Defender for Cloud (security posture)    │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              AZURE STACK HCI OS (on each node)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Hyper-V (VMs) │ S2D (Storage Spaces Direct)             │   │
│  │  SDN (Software Defined Networking — via Windows SDN)     │   │
│  │  Cluster Services (Windows Server Failover Clustering)   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  Storage Spaces Direct (S2D) — pools local NVMe across nodes    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Azure Stack HCI certified hardware (600+ solutions listed)     │
│  Intel Xeon / AMD EPYC │ RAM ECC │ NVMe (S2D) │ 10/25GbE RoCE │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Microsoft-centric enterprises running on-premises HCI with Azure cloud management — hybrid cloud strategy bridging on-premises with Azure services.

## Pros
- Azure portal management for on-premises workloads
- Azure Arc extends cloud services on-premises
- Native Windows + Hyper-V support
- Microsoft support SLA

## Cons
- Microsoft Azure subscription required (ongoing cost)
- Windows-centric — Linux is secondary
- Certified hardware list restricts choices
- Complex licensing (Azure Stack HCI + Windows Server + Arc)