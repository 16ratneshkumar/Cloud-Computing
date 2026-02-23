[← Back to Index](../index.md)

# DEPLOYMENT 26 — Bare Metal → HPE VM Essentials → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│               HPE VM Essentials                │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              HPE VM ESSENTIALS MANAGEMENT                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  HPE VM Essentials Console (web UI)                      │   │
│  │  HPE OneView integration (hardware lifecycle)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│          KVM-BASED HYPERVISOR (HPE managed)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Guest VMs (Windows / Linux)                             │   │
│  │  KVM + libvirt backend managed by HPE tooling            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  HPE ProLiant / Synergy / Alletra (HPE hardware)                │
│  iLO (integrated Lights Out) management                         │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
HPE hardware customers seeking VMware alternative after Broadcom pricing changes.