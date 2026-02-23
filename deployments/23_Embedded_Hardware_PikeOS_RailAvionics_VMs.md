[← Back to Index](../index.md)

# DEPLOYMENT 23 — Embedded Hardware → PikeOS → Rail/Avionics VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│               Rail/Avionics VMs                │
├──────────────────────────────────────────────────┤
│                     PikeOS                     │
├──────────────────────────────────────────────────┤
│               Embedded Hardware                │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌──────────────────────────────────────────────────────────┐
│         SAFETY + GENERAL PURPOSE PARTITIONS              │
│  ┌────────────────────┐  ┌───────────────────────────┐   │
│  │  POSIX RTOS Task   │  │  Linux Guest              │   │
│  │  (EN 50128 SIL 4   │  │  (maintenance/diagnostics)│   │
│  │   rail signaling)  │  │                           │   │
│  └────────────────────┘  └───────────────────────────┘   │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│          PikeOS (SYSGO) — certified partitioning RTOS    │
│  EN 50128 SIL 4 │ DO-178C │ IEC 61508                    │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│     Embedded Hardware (Rail / Avionics)                  │
│  PowerPC / ARM │ Radiation-hardened CPUs possible        │
│                  BARE METAL                              │
└──────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Rail signaling systems (ETCS), avionics (ARINC 653), and defense.