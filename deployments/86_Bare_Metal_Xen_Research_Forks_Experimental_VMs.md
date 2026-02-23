[← Back to Index](../index.md)

# DEPLOYMENT 86 — Bare Metal → Xen Research Forks → Experimental VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                Experimental VMs                │
├──────────────────────────────────────────────────┤
│               Xen Research Forks               │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              EXPERIMENTAL VM WORKLOADS                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  HVM / PV / PVH guests (various virtualization modes)    │   │
│  │  Research: unikernel VMs │ hardened VMs │ micro-VMs      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              XEN RESEARCH FORK / EXTENDED XEN                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Examples:                                               │   │
│  │  XenServer (Citrix commercialization of Xen)             │   │
│  │  Xen on ARM (for mobile/embedded research)               │   │
│  │  XAPI-based toolstack variants                           │   │
│  │  PVH (PV + HVM hybrid) mode research                     │   │
│  │  Dom0-less Xen (no privileged Dom0 — distributed control)│   │
│  │                                                          │   │
│  │  Dom0-less: multiple driver domains, no single Dom0      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              XEN HYPERVISOR (below all OSes)                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  x86 (primary) / ARM (growing) │ RAM │ NIC                    │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Academic research on hypervisor security, embedded Xen (Xilinx Zynq UltraScale+), and Dom0-less designs for car/industrial systems.