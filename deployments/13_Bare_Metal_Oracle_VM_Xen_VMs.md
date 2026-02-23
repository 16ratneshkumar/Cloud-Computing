[← Back to Index](../index.md)

# DEPLOYMENT 13 — Bare Metal → Oracle VM (Xen) → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                Oracle VM (Xen)                 │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              ORACLE VM MANAGER                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Oracle VM Manager (web console / CLI)                   │   │
│  │  Oracle Enterprise Manager integration                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              ORACLE VM SERVER (Xen-based)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Dom0 (Oracle Linux) — control domain                    │   │
│  │  DomU guest VMs:                                         │   │
│  │  ┌─────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │ Oracle Linux VM │  │ Windows Server VM             │  │   │
│  │  │ (Oracle DB)     │  │                               │  │   │
│  │  └─────────────────┘  └──────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  XEN HYPERVISOR                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Oracle-certified hardware │ CPU │ RAM │ SAN/NAS storage        │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Running Oracle Database and Oracle Middleware in Oracle-licensed and Oracle-supported virtualization. Gives Oracle license compliance advantages.

## Pros
- Oracle-supported for Oracle workloads
- Oracle license optimization (CPU counting)
- Xen performance

## Cons
- Oracle-centric; declining market share
- More expensive Oracle support
- Xen base aging vs KVM