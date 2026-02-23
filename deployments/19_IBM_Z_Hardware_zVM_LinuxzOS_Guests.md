[← Back to Index](../index.md)

# DEPLOYMENT 19 — IBM Z Hardware → z/VM → Linux/zOS Guests

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                Linux/zOS Guests                │
├──────────────────────────────────────────────────┤
│                      z/VM                      │
├──────────────────────────────────────────────────┤
│                 IBM Z Hardware                 │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                  TENANT WORKLOADS                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│  │  z/OS      │  │  Linux on  │  │  Linux on  │  │  z/VSE   │  │
│  │  LPAR      │  │  Z (RHEL)  │  │  Z (Ubuntu)│  │  LPAR    │  │
│  │  (COBOL,   │  │  (Java,    │  │  (DB2,     │  │  (legacy)│  │
│  │  DB2, CICS)│  │  Python)   │  │  WebSphere)│  │          │  │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │
│         (thousands of Linux guests possible on one system)      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  z/VM HYPERVISOR                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  z/VM  (CP — Control Program)                            │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────┐  │   │
│  │  │  Virtual Machine│  │  Guest Memory   │  │  DASD   │  │   │
│  │  │  Scheduler (SRM)│  │  Management     │  │  Mgmt   │  │   │
│  │  │                 │  │                 │  │         │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────┘  │   │
│  │  ┌──────────────────────────────────────────────────────┐│   │
│  │  │  CMS (Conversational Monitor System) — admin env    ││   │
│  │  └──────────────────────────────────────────────────────┘│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              IBM Z HARDWARE (z16 / z15 / LinuxONE)              │
│  ┌───────────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │  IBM z CPU        │  │  RAM          │  │  DASD / Flash   │  │
│  │  (S390X arch)     │  │  (up to 32TB  │  │  Express / DS8k │  │
│  │  PU, IFL, ICF,    │  │   on z16)     │  │  storage        │  │
│  │  SAP, zIIP cpus   │  │               │  │                 │  │
│  └───────────────────┘  └───────────────┘  └─────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RAS: Redundant CPUs/memory, chip-spare, fault isolation  │   │
│  │  Pervasive Encryption, Secure Boot                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Banking (SWIFT, ACH transactions), insurance (batch processing), government, and healthcare — where 99.9999% uptime and extremely high VM density are mandatory.

## Pros
- Six nines availability
- Runs thousands of VMs per system
- Hardware memory encryption by default
- Unmatched I/O for transaction processing
- 50+ years of refinement

## Cons
- Extremely expensive (millions USD)
- IBM ecosystem lock-in
- Requires specialized mainframe expertise
- z/OS skills are rare