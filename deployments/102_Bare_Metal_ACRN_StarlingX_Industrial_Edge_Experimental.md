[← Back to Index](../index.md)

# DEPLOYMENT 102 — Bare Metal → ACRN + StarlingX → Industrial Edge (Experimental)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│         Industrial Edge (Experimental)         │
├──────────────────────────────────────────────────┤
│                ACRN + StarlingX                │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MIXED-CRITICALITY EDGE WORKLOADS                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RTOS partition (hard RT — motor control, PLC)           │   │
│  │  Linux/K8s partition (general compute — ML, monitoring)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              STARLINGX (K8s/OpenStack management)               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Manages the general compute partition (Linux side)      │   │
│  │  K8s for cloud-native edge workloads                     │   │
│  │  Coordinates with ACRN for partition lifecycle           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              ACRN HYPERVISOR (Intel IoT/auto type-1)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Service OS (SOS) — Linux, manages UOS lifecycle         │   │
│  │  User OS (UOS) — RT partition for industrial control     │   │
│  │  LAPIC passthrough — native timer interrupts for RT      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Intel VT-x (ACRN requires Intel)
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Intel Edge SoC)               │
│  Intel Atom / Core / Xeon-D │ VT-x │ RAM │ Industrial I/O      │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Intel-backed experimental architecture for autonomous vehicle or factory automation — hard RT workloads + cloud-native management on same edge hardware. **Research/experimental status.**