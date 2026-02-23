[← Back to Index](../index.md)

# DEPLOYMENT 85 — Bare Metal → NOVA Microhypervisor → VMs (Research)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                 VMs (Research)                 │
├──────────────────────────────────────────────────┤
│              NOVA Microhypervisor              │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              GUEST VMs                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Linux guest │ Other OS guests                           │   │
│  │  Runs via NOVA's thin hypervisor abstraction             │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              NOVA MICROHYPERVISOR                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  NOVA: Minimalist Type-1 microhypervisor                 │   │
│  │  Trusted Computing Base (TCB) < 10,000 lines             │   │
│  │  Capability-based access control (like seL4)             │   │
│  │  Runs on Intel VT-x hardware                             │   │
│  │  User-level VMMs (each VM has a user-space VMM process)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (x86)                          │
│  Intel VT-x required │ RAM │ NIC                               │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Security research, trusted computing, and base for Genode OS framework.