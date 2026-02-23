[← Back to Index](../index.md)

# DEPLOYMENT 35 — Bare Metal → KVM → VMs → Docker/LXC (Multi-Tenant)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│           Docker/LXC (Multi-Tenant)            │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                      KVM                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT APPLICATIONS (inside per-tenant VM)         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  TENANT A VM                  TENANT B VM                │   │
│  │  ┌──────────────────────┐     ┌──────────────────────┐   │   │
│  │  │  Docker containers   │     │  LXC containers       │   │   │
│  │  │  ┌────┐ ┌────┐ ┌───┐ │     │  ┌────┐ ┌────┐       │   │   │
│  │  │  │App1│ │App2│ │DB │ │     │  │Web │ │API │       │   │   │
│  │  │  └────┘ └────┘ └───┘ │     │  └────┘ └────┘       │   │   │
│  │  │  docker-compose.yml  │     │  (OS containers)      │   │   │
│  │  └──────────────────────┘     └──────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  each tenant = isolated VM
┌────────────────────────────▼────────────────────────────────────┐
│              KVM VIRTUAL MACHINES (one per tenant)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VM: tenant-A            VM: tenant-B                    │   │
│  │  Ubuntu 22.04            Debian 12                       │   │
│  │  4 vCPU / 8GB RAM        4 vCPU / 8GB RAM               │   │
│  │  Docker Engine installed  LXD installed                  │   │
│  │  veth → virbr0            veth → virbr0                  │   │
│  │  Disk: LVM thin volume   Disk: LVM thin volume          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              VM MANAGEMENT PLATFORM                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Proxmox / oVirt / OpenStack Nova / libvirt              │   │
│  │  → provisions VMs, manages billing, quota enforcement    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU + KVM                                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KVM kernel module │ QEMU processes │ LVM thin pools     │   │
│  │  Linux bridge (virbr0) │ iptables per-tenant firewall    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (HOST)                          │
│  KVM │ cgroups │ namespaces │ iptables │ TUN/TAP               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x) │ RAM │ SSD/NVMe │ 10GbE NIC                      │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Each tenant gets a dedicated VM providing hardware-level isolation
- Inside the VM, tenants run Docker or LXC for application-level container density
- VM boundaries prevent cross-tenant kernel exploits
- The platform manages VM lifecycle, billing, and networking per tenant

## Use Case
VPS and cloud hosting providers, MSPs, and multi-tenant SaaS platforms needing strong tenant isolation with container flexibility inside each tenant's boundary.

## Pros
- Strong VM-level tenant isolation
- Tenants have root in their VM
- Container density within each tenant's VM
- Simple billing model (per VM)

## Cons
- Double overhead: VM + containers
- No cross-VM container orchestration
- Higher per-tenant cost than bare container hosting
- Container sprawl hard to manage