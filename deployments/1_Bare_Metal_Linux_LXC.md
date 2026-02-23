[← Back to Index](../index.md)

# DEPLOYMENT 1 — Bare Metal → Linux → LXC

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)               │
├──────────────────────────────────────────────────┤
│             LXC Container Instance               │
├──────────────────────────────────────────────────┤
│         Host OS (Linux Kernel Stack)             │
├──────────────────────────────────────────────────┤
│            Bare Metal Hardware                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                    WORKLOADS / APPLICATIONS                      │
│   [App Process A]    [App Process B]    [App Process C]          │
└────────────┬────────────────┬───────────────────┬───────────────┘
             │                │                   │
┌────────────▼────────────────▼───────────────────▼───────────────┐
│                    LXC CONTAINERS                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Container A  │  │ Container B  │  │    Container C       │  │
│  │ (own rootfs) │  │ (own rootfs) │  │    (own rootfs)      │  │
│  │ own PID ns   │  │ own PID ns   │  │    own PID ns        │  │
│  │ own net ns   │  │ own net ns   │  │    own net ns        │  │
│  │ own mnt ns   │  │ own mnt ns   │  │    own mnt ns        │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│         Managed via: lxc-start / lxc-attach / lxc-ls            │
└────────────────────────────┬────────────────────────────────────┘
                             │  namespaces + cgroups
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL                                  │
│  ┌────────────┐  ┌─────────────┐  ┌──────────┐  ┌──────────┐  │
│  │ cgroups v2 │  │ namespaces  │  │ seccomp  │  │ AppArmor │  │
│  │(CPU/mem    │  │(pid/net/mnt │  │(syscall  │  │(MAC      │  │
│  │ limits)    │  │ /uts/ipc)   │  │ filter)  │  │ policy)  │  │
│  └────────────┘  └─────────────┘  └──────────┘  └──────────┘  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │        Virtual Network Bridge (lxcbr0 / veth pairs)    │    │
│  └─────────────────────────────────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────────┘
                             │  direct hardware access
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────────┐    │
│  │  CPU (x86/   │  │  RAM (shared  │  │  Storage (disk /  │    │
│  │  ARM)        │  │  by all conts)│  │  NVMe / SAN)      │    │
│  └──────────────┘  └───────────────┘  └───────────────────┘    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           Network Interface Card (NIC / bond)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Utilizes `lxcbr0` for NAT/Bridge. One end of a `veth` pair is in the container, the other on the host bridge.
- **Storage:** Uses the host filesystem directly, often with `ZFS` or `Btrfs` for copy-on-write snapshots.
- **Security:** Relies on Linux Kernel namespaces (PID, Net, Mount) and Cgroups for isolation.
- **Management:** Managed via `lxc` command line or `LXD` API.

## How It Works
- Linux kernel provides **namespaces** (process, network, mount, UTS, IPC) giving each container an isolated view of the system
- **cgroups** enforce CPU, memory, and I/O resource limits per container
- LXC tools (lxc-start, lxc-attach) manage container lifecycle directly without any daemon
- All containers share the **same kernel** — no virtualization overhead

## Use Case
Direct lightweight containerization. Embedded systems, DIY VPS hosting, minimal overhead dev/test environments.

## Pros
- Near-native performance
- No hypervisor overhead
- Fine-grained resource control

## Cons
- Shared kernel security risk
- No live migration
- Manual network setup
