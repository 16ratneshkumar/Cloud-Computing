[← Back to Index](../index.md)

# DEPLOYMENT 18 — Bare Metal → QEMU (Software Emulation, No KVM)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│       QEMU (Software Emulation, No KVM)        │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              CROSS-ARCHITECTURE GUEST VM                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ARM64 Guest OS (on x86 host)                            │   │
│  │  OR: RISC-V Guest │ MIPS Guest │ PowerPC Guest           │   │
│  │  Full software-emulated architecture                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  fully emulated (no hardware virt)
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU (PURE SOFTWARE EMULATION)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  qemu-system-aarch64 / qemu-system-riscv64 / etc.        │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌───────────┐   │   │
│  │  │  TCG Engine    │  │  Device Models │  │ BIOS/UEFI │   │   │
│  │  │  (Tiny Code    │  │  (virtio /     │  │ Firmware  │   │   │
│  │  │   Generator —  │  │  emulated AHCI,│  │           │   │   │
│  │  │  JIT translate │  │  e1000, etc.)  │  │           │   │   │
│  │  │  guest→host    │  │                │  │           │   │   │
│  │  │  instructions) │  │                │  │           │   │   │
│  │  └────────────────┘  └────────────────┘  └───────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  runs as userspace process
┌────────────────────────────▼────────────────────────────────────┐
│                 HOST OS (Linux / macOS / Windows)                │
│   No KVM module needed — pure userspace emulation               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   Any CPU architecture — hardware virtualization NOT required   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Cross-architecture development and testing. Embedded firmware development (testing ARM firmware on x86 workstation), kernel developers testing cross-arch changes, CI pipelines building ARM packages on x86.

## Pros
- Runs on any hardware — no VT-x/AMD-V needed
- Full architecture emulation (ARM, RISC-V, MIPS, SPARC)
- Excellent for embedded development

## Cons
- Very slow (10-100x slower than native)
- Not suitable for production
- High CPU overhead