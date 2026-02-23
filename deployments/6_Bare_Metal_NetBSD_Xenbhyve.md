[← Back to Index](../index.md)

# DEPLOYMENT 6 — Bare Metal → NetBSD → Xen/bhyve

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Xen/bhyve                    │
├──────────────────────────────────────────────────┤
│                     NetBSD                     │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                    GUEST VMs                                    │
│  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │  Xen DomU (PV Guest)   │  │  bhyve VM (HVM Guest)        │  │
│  │  Linux / NetBSD        │  │  Linux / NetBSD              │  │
│  └────────────────────────┘  └──────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│           Xen Dom0 (NetBSD) OR bhyve (NetBSD)                   │
│  ┌──────────────────────────┐  ┌───────────────────────────┐   │
│  │  Xen Dom0 (control)      │  │  bhyve (NetBSD port)      │   │
│  │  NetBSD kernel w/ pv drv │  │  vmm(4) on NetBSD         │   │
│  └──────────────────────────┘  └───────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                     NetBSD HOST KERNEL                          │
│   Xen PV drivers │ virtio drivers │ NPF firewall │ ZFS/FFS     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   CPU (x86 / ARM / SPARC / MIPS) │ RAM │ Storage │ NIC         │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Research and exotic hardware virtualization. NetBSD runs on 100+ platforms; used for porting experiments and academic research.

## Pros
- Extreme hardware portability
- Good for research on non-standard platforms

## Cons
- Very niche — limited production use
- Small community and documentation