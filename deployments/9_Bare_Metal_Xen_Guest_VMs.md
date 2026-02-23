[← Back to Index](../index.md)

# DEPLOYMENT 9 — Bare Metal → Xen → Guest VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Guest VMs                    │
├──────────────────────────────────────────────────┤
│                      Xen                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              UNPRIVILEGED GUEST VMs (DomU)                      │
│  ┌─────────────────────────────┐  ┌────────────────────────┐   │
│  │   DomU (PV Guest)           │  │   DomU (HVM Guest)     │   │
│  │   Paravirtualized Linux      │  │   Windows / unmodified │   │
│  │   xen-netfront (NIC driver) │  │   QEMU device emulation│   │
│  │   xen-blkfront (disk driver)│  │   Hardware virtualized │   │
│  │   Direct hypercalls to Xen  │  │   with VT-x / AMD-V    │   │
│  └─────────────────────────────┘  └────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  event channels / grant tables
┌────────────────────────────▼────────────────────────────────────┐
│              PRIVILEGED DOMAIN (Dom0)                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Linux (or NetBSD) Dom0 — control domain                 │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐ │   │
│  │  │  xl toolstack│  │  xenstored   │  │  xenconsoled   │ │   │
│  │  │  (xl list,   │  │  (config     │  │  (VM console   │ │   │
│  │  │   xl create) │  │   database)  │  │   management)  │ │   │
│  │  └──────────────┘  └──────────────┘  └────────────────┘ │   │
│  │  ┌──────────────┐  ┌──────────────────────────────────┐  │   │
│  │  │  netback     │  │  blkback                         │  │   │
│  │  │  (network    │  │  (disk backend driver —           │  │   │
│  │  │   backend)   │  │   serves disk to DomU)           │  │   │
│  │  └──────────────┘  └──────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  dom0 is privileged via Xen grants
┌────────────────────────────▼────────────────────────────────────┐
│                  XEN HYPERVISOR                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Xen runs BELOW the OS — loaded by bootloader (GRUB)     │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌───────────┐   │   │
│  │  │  vCPU Scheduler│  │  Memory Mgmt   │  │  Event    │   │   │
│  │  │  (credit2)     │  │  (ballooning,  │  │  Channels │   │   │
│  │  │                │  │   grant tables)│  │           │   │   │
│  │  └────────────────┘  └────────────────┘  └───────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Xen loaded before any OS
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  ┌──────────────────┐  ┌────────────────┐  ┌────────────────┐  │
│  │  CPU (VT-x/AMD-V │  │  RAM           │  │  Disk / SAN    │  │
│  │  for HVM, plain  │  │                │  │                │  │
│  │  CPU for PV)     │  │                │  │                │  │
│  └──────────────────┘  └────────────────┘  └────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  NIC (Dom0 owns physical NIC → bridges to DomU vifs)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Xen loads before any OS via GRUB — it sits directly on hardware
- Dom0 (privileged Linux) boots first and manages all hardware drivers
- DomU guests communicate with Dom0 via paravirtual drivers (event channels, grant tables, shared memory)
- PV guests make hypercalls to Xen directly; HVM guests use hardware virtualization + QEMU emulation
- Dom0's netback/blkback drivers handle I/O on behalf of guest VMs

## Use Case
Type-1 hypervisor for hosting providers, security-sensitive deployments, and historically AWS EC2. Strong isolation between VMs.

## Pros
- True Type-1 — runs before any OS
- Strong isolation — Dom0 is separate from guests
- Mature — 20+ years of production use
- PV paravirtualization offers very good I/O performance

## Cons
- Dom0 is single point of control (if Dom0 fails, all VMs impacted)
- More complex architecture than KVM
- Declining adoption vs KVM
- PV guests need Xen-aware kernels