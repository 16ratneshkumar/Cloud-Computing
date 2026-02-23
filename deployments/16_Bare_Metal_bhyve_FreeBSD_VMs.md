[← Back to Index](../index.md)

# DEPLOYMENT 16 — Bare Metal → bhyve (FreeBSD) → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                bhyve (FreeBSD)                 │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                    GUEST VIRTUAL MACHINES                       │
│  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │   Linux Guest          │  │   Windows Guest              │  │
│  │   virtio-net           │  │   AHCI / e1000 emulation     │  │
│  │   virtio-blk           │  │   VirtIO drivers (optional)  │  │
│  └────────────────────────┘  └──────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│         BHYVE + VM MANAGEMENT                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  bhyve (userspace VMM process per VM)                    │   │
│  │  bhyvectl — VM management CLI                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────┐  ┌──────────────────────────────────┐    │
│  │  vm-bhyve         │  │  CBSD                           │    │
│  │  (shell mgmt tool)│  │  (management framework)         │    │
│  └──────────────────┘  └──────────────────────────────────┘    │
│  TrueNAS SCALE uses bhyve for VM hosting alongside ZFS          │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/vmm
┌────────────────────────────▼────────────────────────────────────┐
│                FREEBSD KERNEL (with vmm.ko)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vmm.ko (VMX/SVM support) │ ZFS │ pf │ jails            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   Intel VT-x / AMD-V CPU │ RAM │ ZFS Pool (NVMe/HDD) │ NIC     │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
BSD-native hypervisor. Used in TrueNAS SCALE (NAS + VMs on same hardware), pfSense/OPNsense environments, and FreeBSD development shops.

## Pros
- Native FreeBSD — ZFS + pf + bhyve in one OS
- TrueNAS SCALE provides turnkey NAS + VMs
- Good for homelab NAS with VMs

## Cons
- FreeBSD ecosystem only
- Windows support needs extra drivers
- Less mature than KVM