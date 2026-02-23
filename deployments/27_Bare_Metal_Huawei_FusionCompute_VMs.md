[← Back to Index](../index.md)

# DEPLOYMENT 27 — Bare Metal → Huawei FusionCompute → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│              Huawei FusionCompute              │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              FUSIONMANAGER / FUSIONSPHERE                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  FusionManager (cluster management)                      │   │
│  │  FusionSphere OpenStack (cloud layer, optional)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              FUSIONCOMPUTE (CNA + VRM)                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CNA (Compute Node Agent) — KVM hypervisor per node      │   │
│  │  VRM (Virtual Resource Manager) — cluster manager        │   │
│  │  Guest VMs (Windows / Linux / Euler OS)                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Huawei TaiShan (ARM Kunpeng) / FusionServer (x86)              │
│  Huawei OceanStor storage │ CloudEngine network switches        │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Chinese domestic market enterprise cloud. Banks, telecoms, government in China.