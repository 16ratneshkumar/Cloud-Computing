[← Back to Index](../index.md)

# DEPLOYMENT 56 — Bare Metal → OpenNebula → KVM / LXD / VMware → VMs + Containers

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                VMs + Containers                │
├──────────────────────────────────────────────────┤
│               KVM / LXD / VMware               │
├──────────────────────────────────────────────────┤
│                   OpenNebula                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT CLIENTS                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Sunstone (web UI)  │  onevm CLI  │  Terraform provider  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENNEBULA FRONT-END                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  oned (OpenNebula daemon — core)                         │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Scheduler │ ACL Manager │ Image/Network Manager   │  │   │
│  │  │  VM lifecycle │ Cluster management                 │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  SQLite / MySQL (state DB)                               │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  ONE driver to each host
┌────────────────────────────▼────────────────────────────────────┐
│              HYPERVISOR HOSTS (via SSH-based driver)            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  KVM Host        │  │  LXD Host        │  │  VMware ESXi │  │
│  │  (kvm driver)    │  │  (lxd driver)    │  │  (vmware drv)│  │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────┐  │  │
│  │  │ VMs        │  │  │  │ Containers │  │  │  │ VMs    │  │  │
│  │  │ (QEMU/KVM) │  │  │  │ (LXC)     │  │  │  │        │  │  │
│  │  └────────────┘  │  │  └────────────┘  │  │  └────────┘  │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Hypervisor nodes │ Shared storage (NFS/Ceph/iSCSI) │ Network  │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Lightweight private cloud — simpler than OpenStack. Used by European research clouds, universities, and mid-size enterprises.

## Pros
- Much simpler than OpenStack to deploy/operate
- Multi-hypervisor (KVM, LXD, VMware, Firecracker)
- Flexible VM templates
- Good edge cloud support

## Cons
- Smaller community than OpenStack
- Less feature-rich for very large deployments
- Limited RBAC compared to OpenStack/K8s