[← Back to Index](../index.md)

# DEPLOYMENT 21 — Embedded Hardware → QNX Hypervisor → RTOS + Linux

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                  RTOS + Linux                  │
├──────────────────────────────────────────────────┤
│                 QNX Hypervisor                 │
├──────────────────────────────────────────────────┤
│               Embedded Hardware                │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│           MIXED-CRITICALITY GUEST PARTITIONS                    │
│  ┌────────────────────────────┐  ┌───────────────────────────┐  │
│  │  SAFETY PARTITION (RT)     │  │  GENERAL PURPOSE PARTITION│  │
│  │  QNX RTOS                  │  │  Linux / Android          │  │
│  │  ┌──────────┐  ┌────────┐  │  │  ┌───────────┐  ┌──────┐ │  │
│  │  │ Brake    │  │ Power  │  │  │  │Infotainment│  │  OTA │ │  │
│  │  │ Control  │  │ Mgmt   │  │  │  │ (Android)  │  │Update│ │  │
│  │  └──────────┘  └────────┘  │  │  └───────────┘  └──────┘ │  │
│  │  Hard RT guarantees        │  │  Soft RT / best-effort    │  │
│  │  ISO 26262 ASIL-D          │  │  No safety cert required  │  │
│  └────────────────────────────┘  └───────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │  inter-partition communication
┌────────────────────────────▼────────────────────────────────────┐
│              QNX HYPERVISOR                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Temporal Isolation (CPU time guaranteed per partition)   │   │
│  │  Spatial Isolation (memory pages protected per partition) │   │
│  │  Virtual device model (virtio for IPC between partitions) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              EMBEDDED SoC (Intel / ARM / Qualcomm)              │
│  ┌────────────────┐  ┌────────────────┐  ┌───────────────────┐  │
│  │  CPU cores     │  │  RAM           │  │  Peripherals      │  │
│  │  (multi-core   │  │  (partitioned  │  │  (CAN bus, LIN,  │  │
│  │   heterogeneous│  │   between      │  │   PCIe, MIPI,    │  │
│  │   optional)    │  │   partitions)  │  │   Ethernet)       │  │
│  └────────────────┘  └────────────────┘  └───────────────────┘  │
│                  BARE METAL (SoC)                                │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Automotive (digital instrument clusters, ADAS), industrial automation, medical devices requiring mixed-criticality — real-time safety alongside general-purpose Linux.

## Pros
- Hard RT guarantees for safety partition
- ISO 26262 ASIL-D certified
- Temporal and spatial isolation proven in automotive OEMs

## Cons
- Expensive QNX licensing
- Specialized expertise required
- Closed ecosystem