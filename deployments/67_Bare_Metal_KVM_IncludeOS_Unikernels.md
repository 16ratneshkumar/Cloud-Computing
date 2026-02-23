[← Back to Index](../index.md)

# DEPLOYMENT 67 — Bare Metal → KVM → IncludeOS Unikernels

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│              IncludeOS Unikernels              │
├──────────────────────────────────────────────────┤
│                      KVM                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              UNIKERNEL SERVICE                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  C++ Application compiled with IncludeOS                 │   │
│  │  → single bootable image (includes OS + app)             │   │
│  │  → runs as KVM guest                                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU + KVM                                         │
│  qemu-system-x86_64 -enable-kvm -kernel includeos.img          │
│  virtio-net │ virtio-blk │ minimal QEMU config                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (host)                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
High-performance network functions and security-sensitive services requiring C++ ecosystem. NFV research and specialized cloud services.