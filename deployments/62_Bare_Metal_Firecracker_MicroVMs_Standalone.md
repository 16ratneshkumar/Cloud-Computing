[← Back to Index](../index.md)

# DEPLOYMENT 62 — Bare Metal → Firecracker → MicroVMs (Standalone)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│             MicroVMs (Standalone)              │
├──────────────────────────────────────────────────┤
│                  Firecracker                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS (per-MicroVM isolation)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Lambda Function A  │  Lambda Function B  │  Function C  │   │
│  │  (Python 3.11)      │  (Node.js 20)       │  (Go 1.21)   │   │
│  │  Own Linux kernel   │  Own Linux kernel   │  Own kernel  │   │
│  │  Own rootfs         │  Own rootfs         │  Own rootfs  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  one Firecracker process per MicroVM
┌────────────────────────────▼────────────────────────────────────┐
│              FIRECRACKER VMM LAYER                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  jailer (security sandbox per Firecracker process)       │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  chroot (isolates Firecracker binary + config)     │  │   │
│  │  │  cgroup (limits CPU/mem for this MicroVM)          │  │   │
│  │  │  seccomp (syscall filter — strict allowlist)       │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  firecracker binary (Rust VMM ~1MB)          │  │  │   │
│  │  │  │  REST API on Unix socket (config + control)  │  │  │   │
│  │  │  │  ┌────────────────────────────────────────┐  │  │  │   │
│  │  │  │  │  MicroVM:                              │  │  │  │   │
│  │  │  │  │  virtio-net (TAP device)               │  │  │  │   │
│  │  │  │  │  virtio-blk (block device or mem)      │  │  │  │   │
│  │  │  │  │  Minimal Linux kernel (5.10 LTS)       │  │  │  │   │
│  │  │  │  │  rootfs (snapshot-based, copy-on-write)│  │  │  │   │
│  │  │  │  │  MMIO devices ONLY (no PCI)            │  │  │  │   │
│  │  │  │  └────────────────────────────────────────┘  │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  [Thousands of Firecracker processes per host]           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/kvm (ioctls)
┌────────────────────────────▼────────────────────────────────────┐
│              KVM KERNEL MODULE                                  │
│  kvm.ko + kvm-intel.ko / kvm-amd.ko                            │
│  EPT (Extended Page Tables) for guest memory isolation          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (host)                          │
│  cgroups v2 (per jailer) │ TUN/TAP │ seccomp │ iptables        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU: Intel VT-x or AMD-V (mandatory) + AMD64                  │
│  RAM: High density (1000 MicroVMs × 128MB = 128GB+)             │
│  NVMe: Fast local storage for rootfs snapshots                  │
│  NIC: 10GbE+ (many TAP devices, one per MicroVM)               │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- jailer creates a chroot + cgroup + seccomp sandbox for each Firecracker process
- Firecracker is configured via REST API calls on its Unix socket before VM boot
- VM gets minimal Linux kernel + rootfs (read-only snapshot, copy-on-write per VM)
- virtio-net uses a host TAP device for networking
- Boot: ~125ms; Memory per VMM: <5MB; allows thousands per host
- AWS Lambda runs each function invocation in a Firecracker MicroVM — this is Firecracker's primary production deployment

## Use Case
Serverless/FaaS platforms, multi-tenant code execution, and any scenario needing fast startup + strong isolation at scale. AWS Lambda, AWS Fargate internals.

## Pros
- Sub-second startup (~125ms cold start)
- <5MB VMM overhead per MicroVM
- KVM hardware isolation
- Open source (Apache 2.0, by AWS)
- Rust — memory-safe

## Cons
- Minimal device support (intentional)
- Linux guests only (x86/ARM64)
- No GPU passthrough
- Must build orchestration layer on top
- Requires KVM hardware support