[← Back to Index](../index.md)

# DEPLOYMENT 40 — Bare Metal → Kubernetes → Firecracker MicroVMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│              Firecracker MicroVMs              │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              SERVERLESS / FAAS WORKLOADS                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Function A  │  Function B  │  Function C                │   │
│  │  (Python)    │  (Node.js)   │  (Go)                      │   │
│  │  Each in own MicroVM — hardware isolated                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + containerd                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RuntimeClass: kata-fc  →  Kata + Firecracker backend    │   │
│  │  OR: direct Firecracker integration (e.g., Weaveworks)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  one Firecracker VMM per pod
┌────────────────────────────▼────────────────────────────────────┐
│              FIRECRACKER MicroVM Manager                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  firecracker binary (Rust, ~1MB)                         │   │
│  │  REST API on Unix socket → management interface          │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Per-Workload MicroVM:                           │    │   │
│  │  │  ┌────────────────────────────────────────────┐  │    │   │
│  │  │  │  Minimal Linux kernel (~5.10 LTS)          │  │    │   │
│  │  │  │  rootfs (read-only, shared or per-VM)      │  │    │   │
│  │  │  │  virtio-net (TAP device)                   │  │    │   │
│  │  │  │  virtio-blk (block device)                 │  │    │   │
│  │  │  │  NO: USB, PCI hotplug, BIOS emulation      │  │    │   │
│  │  │  │  → intentionally minimal                   │  │    │   │
│  │  │  │  Boot time: ~125ms                         │  │    │   │
│  │  │  │  Overhead: <5MB per VMM process            │  │    │   │
│  │  │  └────────────────────────────────────────────┘  │    │   │
│  │  └──────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────┘   │
│  jailer process (sandboxes Firecracker — seccomp + chroot)   │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/kvm
┌────────────────────────────▼────────────────────────────────────┐
│              KVM KERNEL MODULE                                  │
│  kvm.ko + kvm-intel/amd.ko → hardware VT-x/AMD-V               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (host)                          │
│  KVM │ TUN/TAP (virtio-net) │ cgroups (jailer enforcement)      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (Intel VT-x or AMD-V + AMD64)                             │
│  RAM │ NVMe │ NIC (10GbE+)                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Firecracker is a minimal VMM written in Rust, purposely removing all non-essential device emulation
- Each microVM is started by the Firecracker binary, configured via a REST API on a Unix socket
- The jailer process sandboxes Firecracker using cgroups, seccomp, and a chroot jail
- virtio-net uses a TAP device on the host for networking
- Multiple microVMs share a read-only rootfs (snapshot) — only dirty pages are per-VM
- Boot time: ~125ms; memory overhead per VMM: <5MB

## Use Case
AWS Lambda internals, self-hosted FaaS, and any scenario needing thousands of secure VM-per-request workloads with minimal overhead.

## Pros
- Sub-second startup (~125ms)
- <5MB memory overhead per VMM
- KVM hardware isolation
- Rust — memory-safe implementation
- Open source (Apache 2.0)

## Cons
- Minimal device support (intentionally limited)
- Linux guests only (x86, ARM64)
- Must build orchestration layer on top
- No storage hot-plug or USB
- Requires KVM hardware support