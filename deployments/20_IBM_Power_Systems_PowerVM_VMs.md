[← Back to Index](../index.md)

# DEPLOYMENT 20 — IBM Power Systems → PowerVM → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                    PowerVM                     │
├──────────────────────────────────────────────────┤
│               IBM Power Systems                │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOGICAL PARTITIONS (LPARs)                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐   │
│  │  AIX LPAR   │  │  Linux LPAR │  │  IBM i LPAR          │   │
│  │  (Oracle DB,│  │  (SAP HANA, │  │  (AS/400 workloads)  │   │
│  │   WebSphere)│  │   Red Hat)  │  │                      │   │
│  └─────────────┘  └─────────────┘  └──────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  PowerVM HYPERVISOR                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  POWER Hypervisor (PHYP) — embedded in firmware          │   │
│  │  ┌──────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │  Micro-partitions│  │  Shared Processor Pools      │  │   │
│  │  │  (0.1 vCPU       │  │  (dynamic CPU allocation)    │  │   │
│  │  │  granularity)    │  │                              │  │   │
│  │  └──────────────────┘  └──────────────────────────────┘  │   │
│  │  ┌──────────────────────────────────────────────────────┐ │   │
│  │  │  Virtual I/O Server (VIOS) — storage/net for LPARs  │ │   │
│  │  └──────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│  HMC (Hardware Management Console) — external management node   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              IBM POWER HARDWARE (POWER9/POWER10)                │
│  ┌──────────────────┐  ┌────────────────┐  ┌────────────────┐  │
│  │  POWER CPU       │  │  RAM (up to    │  │  NVMe / SAN    │  │
│  │  (POWER10 arch)  │  │  8TB per node) │  │  storage       │  │
│  └──────────────────┘  └────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
SAP HANA, Oracle Database, and AIX mission-critical workloads requiring POWER architecture's memory bandwidth and RAS.

## Pros
- Superior memory bandwidth for in-memory databases
- AIX and IBM i native support
- Micro-partitioning (0.1 vCPU granularity)

## Cons
- Very expensive IBM POWER hardware
- Specialized expertise scarce