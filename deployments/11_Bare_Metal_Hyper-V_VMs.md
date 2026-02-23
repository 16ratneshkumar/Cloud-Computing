[← Back to Index](../index.md)

# DEPLOYMENT 11 — Bare Metal → Hyper-V → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                    Hyper-V                     │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT LAYER                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  Windows Admin   │  │  Azure Arc       │  │  System      │  │
│  │  Center (WAC)    │  │  (Azure portal   │  │  Center VMM  │  │
│  │  (web UI)        │  │   management)    │  │  (SCVMM)     │  │
│  └──────────┬───────┘  └────────┬─────────┘  └───────┬──────┘  │
└─────────────┼───────────────────┼────────────────────┼─────────┘
              └───────────────────┼────────────────────┘
                                  │  WMI / PowerShell / WinRM
┌─────────────────────────────────▼───────────────────────────────┐
│                   PARENT PARTITION (Management OS)               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Windows Server 2022 (Parent Partition)                  │   │
│  │  Has privileged access to all hardware via Hyper-V VSP   │   │
│  │  ┌──────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │  Hyper-V Manager │  │  PowerShell Hyper-V module   │  │   │
│  │  │  (GUI)           │  │  (Get-VM, New-VM, etc.)      │  │   │
│  │  └──────────────────┘  └──────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │         CHILD VIRTUAL MACHINES (Guest VMs)               │   │
│  │  ┌───────────────────┐  ┌───────────────────────────┐   │   │
│  │  │  Windows Server VM│  │  Linux VM (Gen 2)         │   │   │
│  │  │  Gen 1 / Gen 2    │  │  Hyper-V integration svcs │   │   │
│  │  │  Synthetic adapters│  │ virtio / Hyper-V drivers  │   │   │
│  │  └───────────────────┘  └───────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  enlightened I/O via VMBus
┌────────────────────────────▼────────────────────────────────────┐
│                  HYPER-V HYPERVISOR                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Type-1 — loads before Windows via bootloader            │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────┐  │   │
│  │  │  VMBus          │  │  vCPU Scheduler │  │  Memory │  │   │
│  │  │  (fast I/O bus  │  │                 │  │  Mgmt   │  │   │
│  │  │   Parent↔Child) │  │                 │  │         │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Intel Xeon / AMD EPYC │ RAM │ Storage │ 10GbE/25GbE NIC       │
│  (Azure Stack HCI certified hardware for HCI deployments)       │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Hyper-V is a Type-1 hypervisor loaded by the Windows bootloader before Windows starts
- Windows Server runs in the "Parent Partition" — a privileged VM with hardware access
- Child VMs (guests) communicate via VMBus — a high-speed paravirtual bus
- Synthetic (enlightened) devices replace emulated hardware for better performance
- Azure Arc extends Azure management to on-premises Hyper-V VMs

## Use Case
Windows-centric enterprises, Active Directory environments, SQL Server clusters, and organizations leveraging Azure hybrid cloud (Azure Arc, Azure Stack HCI).

## Pros
- Free with Windows Server licensing
- Excellent Windows workload support
- Azure Arc hybrid management
- PowerShell automation-friendly

## Cons
- Windows-centric — Linux is secondary
- GUI cluster management requires SCVMM (additional cost)
- Less flexible than KVM for pure Linux workloads