[← Back to Index](../index.md)

# DEPLOYMENT 10 — Bare Metal → VMware ESXi → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                  VMware ESXi                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT LAYER                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vCenter Server (separate appliance/VM)                  │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │   │
│  │  │  vSphere    │  │  vMotion     │  │  DRS            │ │   │
│  │  │  Web Client │  │  (live VM    │  │  (auto workload │ │   │
│  │  │  (HTML5)    │  │   migration) │  │   balancing)    │ │   │
│  │  └─────────────┘  └──────────────┘  └─────────────────┘ │   │
│  │  ┌──────────────┐  ┌──────────────────────────────────┐  │   │
│  │  │  HA (High    │  │  vSAN (virtual SAN storage)      │  │   │
│  │  │  Availability│  │  NSX (software-defined network)  │  │   │
│  │  └──────────────┘  └──────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  manages multiple ESXi hosts
┌────────────────────────────▼────────────────────────────────────┐
│                  VMware ESXi HOST                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                 GUEST VMs                                │   │
│  │  ┌─────────────────┐  ┌─────────────────┐              │   │
│  │  │ Windows Server  │  │ Linux VM         │              │   │
│  │  │ VMware Tools    │  │ open-vm-tools    │              │   │
│  │  │ VMXNET3 (NIC)   │  │ VMXNET3 (NIC)   │              │   │
│  │  │ PVSCSI (disk)   │  │ PVSCSI (disk)   │              │   │
│  │  └─────────────────┘  └─────────────────┘              │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VMkernel (ESXi Kernel)                                  │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌───────────┐  │   │
│  │  │  CPU Scheduler │  │  Memory Mgmt   │  │  Storage  │  │   │
│  │  │  (resource     │  │  (TPS, balloon,│  │  Stack    │  │   │
│  │  │   pools)       │  │  swap)         │  │  (vmfs)   │  │   │
│  │  └────────────────┘  └────────────────┘  └───────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Virtual Switch (vSwitch / Distributed vSwitch)   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  VMware-certified hardware (HCL)                                │
│  CPU (Intel Xeon / AMD EPYC) │ RAM │ VMFS Datastore │ 10GbE NIC│
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- ESXi (VMkernel) installs directly on bare metal and provides VMs to the management layer
- VMware Tools / open-vm-tools inside VMs provide paravirtual drivers and guest integration
- vCenter manages multiple ESXi hosts, enabling vMotion (live migration), DRS (load balancing), and HA (failover)
- vSAN pools local disks across ESXi hosts into a shared distributed datastore
- NSX provides software-defined networking across the cluster

## Use Case
Dominant enterprise VM platform. Windows workloads, Oracle DB, SAP. Enterprises needing mature HA, live migration, and full commercial support.

## Pros
- Most feature-rich enterprise hypervisor
- Excellent Windows workload support
- vMotion for zero-downtime maintenance
- DRS auto-balances VM placement
- Enterprise support and compliance certifications

## Cons
- Very expensive licensing (especially post-Broadcom 2024)
- Closed source — no community contribution
- Vendor lock-in
- Broadcom's licensing changes have driven many migrations away