[← Back to Index](../index.md)

# DEPLOYMENT 68 — Bare Metal → KVM → OSv Unikernels

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                 OSv Unikernels                 │
├──────────────────────────────────────────────────┤
│                      KVM                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              JVM / POSIX APPLICATION                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Java / Scala / Clojure / C / Ruby application           │   │
│  │  runs on OSv kernel (POSIX compatibility layer)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  OSv replaces OS for single application
┌────────────────────────────▼────────────────────────────────────┐
│              OSv UNIKERNEL (bootable VM image)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Application (Java JAR / native binary)                  │   │
│  │  JVM or language runtime (embedded)                      │   │
│  │  OSv kernel (POSIX subset implementation)                │   │
│  │  virtio-net │ virtio-blk │ minimal device support        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU + KVM                                         │
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
Running JVM applications (Spring Boot, etc.) as unikernels for cloud workloads — eliminates full OS overhead for JVM services.