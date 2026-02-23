[← Back to Index](../index.md)

# DEPLOYMENT 28 — Bare Metal → Scale Computing HC3 → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│              Scale Computing HC3               │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              SCALE COMPUTING MANAGEMENT                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  HyperCore UI (web management)                           │   │
│  │  REST API │ Scale Computing Fleet Manager (multi-site)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              HyperCore OS (KVM-based HCI)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KVM Virtual Machines (Windows / Linux)                  │   │
│  │  SCRIBE (distributed storage — local NVMe pool)          │   │
│  │  Self-healing storage (auto-rebuild on node failure)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Scale Computing HC3/HC4 Edge Nodes (3-node minimum for prod)  │
│  Intel CPU │ RAM │ NVMe SSD │ 10GbE NIC                        │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Edge computing and ROBO (remote office/branch office) virtualization. Simple HCI for non-IT environments.