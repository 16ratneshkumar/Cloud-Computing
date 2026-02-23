[← Back to Index](../index.md)

# DEPLOYMENT 24 — Bare Metal → ACRN Hypervisor → Service VM + RT VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│              Service VM + RT VMs               │
├──────────────────────────────────────────────────┤
│                ACRN Hypervisor                 │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                  GUEST PARTITIONS                               │
│  ┌────────────────────────────┐  ┌───────────────────────────┐  │
│  │  SERVICE VM (SOS)          │  │  USER VMs (UOS)           │  │
│  │  Ubuntu / Clear Linux      │  │  ┌───────────┐  ┌──────┐  │  │
│  │  ┌─────────────────────┐   │  │  │ RT Linux  │  │ IoT  │  │  │
│  │  │ Device Model (DM)   │   │  │  │ (hard RT  │  │ Work │  │  │
│  │  │ virtio backend devs │   │  │  │ workload) │  │ load │  │  │
│  │  └─────────────────────┘   │  │  └───────────┘  └──────┘  │  │
│  │  Manages UOS lifecycle     │  │                           │  │
│  └────────────────────────────┘  └───────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │  ACRN hypervisor calls
┌────────────────────────────▼────────────────────────────────────┐
│                  ACRN HYPERVISOR                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VMX-based Type-1 hypervisor (Intel VT-x only)           │   │
│  │  ┌──────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │  vCPU Scheduler  │  │  Memory Partitioning         │  │   │
│  │  │  (LAPIC passthru │  │  (EPT page tables)           │  │   │
│  │  │  for RT VMs)     │  │                              │  │   │
│  │  └──────────────────┘  └──────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  ACRN loaded by UEFI / bootloader
┌────────────────────────────▼────────────────────────────────────┐
│              INTEL EDGE HARDWARE (x86 IoT / Automotive SoC)    │
│  Intel VT-x required │ RAM │ eMMC / NVMe │ CAN / LTE / Ethernet│
│                  BARE METAL                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Intel IoT edge devices, connected vehicles (non-safety-critical), industrial PCs requiring mixed real-time and general-purpose workloads with open-source tooling.

## Pros
- Open source (Linux Foundation)
- Designed for Intel edge silicon
- Mixed-criticality support
- Smaller footprint than server hypervisors

## Cons
- Intel x86 only
- Smaller community than KVM
- Not for general-purpose cloud use