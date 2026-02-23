[← Back to Index](../index.md)

# DEPLOYMENT 37 — Bare Metal → LXD → Containers + VMs (Unified)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│           Containers + VMs (Unified)           │
├──────────────────────────────────────────────────┤
│                      LXD                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              UNIFIED WORKLOADS (via LXD)                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  LXC SYSTEM CONTAINERS         LXD VIRTUAL MACHINES     │   │
│  │  ┌──────────────────────┐      ┌──────────────────────┐  │   │
│  │  │ container: web-01    │      │ vm: windows-01       │  │   │
│  │  │ Ubuntu 22.04 LXC     │      │ Windows Server 2022  │  │   │
│  │  │ Nginx                │      │ (QEMU internally)    │  │   │
│  │  │ veth → lxdbr0        │      │ virtio-net           │  │   │
│  │  │ ZFS dataset          │      │ virtio-blk           │  │   │
│  │  └──────────────────────┘      │ UEFI / Secure Boot   │  │   │
│  │  ┌──────────────────────┐      └──────────────────────┘  │   │
│  │  │ container: db-01     │      ┌──────────────────────┐  │   │
│  │  │ PostgreSQL            │      │ vm: legacy-app-01    │  │   │
│  │  │                      │      │ CentOS 7 (legacy)    │  │   │
│  │  └──────────────────────┘      │ (QEMU internally)    │  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  single unified API
┌────────────────────────────▼────────────────────────────────────┐
│              LXD DAEMON (unified management)                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  REST API (:8443)                                        │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Container driver (LXC backend)                  │    │   │
│  │  │  VM driver (QEMU backend — uses qemu-kvm)        │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  │  ┌─────────────┐  ┌────────────────┐  ┌──────────────┐  │   │
│  │  │  ZFS storage│  │  OVN networking│  │  Profiles    │  │   │
│  │  │  pool        │  │  (L3 routing)  │  │  (config     │  │   │
│  │  │  (shared)    │  │                │  │   templates) │  │   │
│  │  └─────────────┘  └────────────────┘  └──────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KERNEL LAYER                                       │
│  ┌─────────────────────┐  ┌──────────────────────────────────┐  │
│  │  LXC (namespaces +  │  │  KVM + QEMU (for LXD VMs)        │  │
│  │  cgroups for conts) │  │  VMX/SVM + EPT                   │  │
│  └─────────────────────┘  └──────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V for VMs) │ RAM │ NVMe (ZFS pool) │ NIC        │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
SMBs and hosters needing both Linux containers (for efficient Linux apps) and full VMs (for Windows or legacy systems) from a single unified API without Proxmox.

## Pros
- One API, one CLI for both containers and VMs
- ZFS provides excellent storage for both types
- OVN provides advanced networking
- Snapshots and migration for both

## Cons
- LXD VM feature is less mature than Proxmox KVM
- Canonical ecosystem with community fork tension
- Not cloud-native