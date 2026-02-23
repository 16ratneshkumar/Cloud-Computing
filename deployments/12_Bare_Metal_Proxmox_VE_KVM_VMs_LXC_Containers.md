[← Back to Index](../index.md)

# DEPLOYMENT 12 — Bare Metal → Proxmox VE → KVM VMs + LXC Containers

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│            KVM VMs + LXC Containers            │
├──────────────────────────────────────────────────┤
│                   Proxmox VE                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                  MANAGEMENT LAYER                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Proxmox Web UI  (port 8006, HTTPS)                      │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │   │
│  │  │  VM/CT Mgmt │  │  Cluster &   │  │  Storage &      │ │   │
│  │  │  (create,   │  │  HA Manager  │  │  Backup Mgmt    │ │   │
│  │  │  snapshots) │  │              │  │  (PBS, NFS,     │ │   │
│  │  │             │  │              │  │   Ceph, ZFS)    │ │   │
│  │  └─────────────┘  └──────────────┘  └─────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────┐  ┌───────────────────────────────┐    │
│  │  pvesh (REST API CLI)│  │  Terraform Proxmox provider   │    │
│  └──────────────────────┘  └───────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│           PROXMOX VE  (Debian 12-based OS)                      │
│                                                                  │
│  ┌─────────────────────────────────┐                            │
│  │        KVM VIRTUAL MACHINES     │                            │
│  │  ┌─────────────┐  ┌──────────┐  │                            │
│  │  │ VM 100      │  │  VM 101  │  │                            │
│  │  │ Windows 11  │  │  Ubuntu  │  │                            │
│  │  │ 8vCPU/32GB  │  │  4vCPU/  │  │                            │
│  │  │ VirtIO NIC  │  │  16GB    │  │                            │
│  │  │ VirtIO SCSI │  │  VirtIO  │  │                            │
│  │  └─────────────┘  └──────────┘  │                            │
│  └─────────────────────────────────┘                            │
│  ┌─────────────────────────────────┐                            │
│  │        LXC CONTAINERS           │                            │
│  │  ┌─────────────┐  ┌──────────┐  │                            │
│  │  │ CT 200      │  │  CT 201  │  │                            │
│  │  │ Nginx       │  │  MariaDB │  │                            │
│  │  │ 2 core/2GB  │  │  2 core/ │  │                            │
│  │  │ veth→bridge │  │  4GB     │  │                            │
│  │  └─────────────┘  └──────────┘  │                            │
│  └─────────────────────────────────┘                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              STORAGE SUBSYSTEM                           │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐   │   │
│  │  │  ZFS     │  │  LVM-thin│  │  Ceph RBD│  │  NFS/  │   │   │
│  │  │  (local) │  │  (local) │  │  (cluster│  │  CIFS  │   │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              NETWORK SUBSYSTEM                           │   │
│  │  Linux bridge (vmbr0) │ VLAN-aware bridge │ OVS         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LXC KERNEL LAYER                             │
│  kvm.ko + kvm-intel/amd.ko │ cgroups v2 │ namespaces │ ZFS      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V) │ RAM (ECC recommended) │ SSD/NVMe │ NIC      │
└─────────────────────────────────────────────────────────────────┘

PROXMOX CLUSTER (3 nodes for HA):
┌──────────────┐    corosync/    ┌──────────────┐    ┌──────────────┐
│  PVE Node 1  │◄──────────────►│  PVE Node 2  │◄──►│  PVE Node 3  │
│  (votes: 1)  │   cluster sync  │  (votes: 1)  │    │  (votes: 1)  │
└──────────────┘                └──────────────┘    └──────────────┘
        │                               │                   │
        └───────────────────────────────┴───────────────────┘
                              Shared Storage (Ceph / NFS / iSCSI)
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Proxmox VE is a Debian-based OS with KVM and LXC management on top
- VMs managed via QEMU/KVM with virtio paravirtual drivers
- Containers managed via LXC with proxmox pct tool
- Clustering uses Corosync (distributed messaging) + pmxcfs (cluster filesystem)
- HA manager monitors VMs and restarts them on other nodes if a host fails
- Ceph can be deployed directly within Proxmox cluster for distributed storage

## Use Case
SMB, homelabs, VMware migration. Full-featured open-source hypervisor with excellent GUI. Most popular VMware alternative in 2024 after Broadcom pricing changes.

## Pros
- Free and open source (paid enterprise support optional)
- Unified VM + container management
- Excellent web UI
- Integrated Ceph for HCI
- Built-in backup (Proxmox Backup Server)
- Active community

## Cons
- Not as enterprise-polished as VMware/Nutanix
- Ceph integration adds significant complexity
- Web UI can feel limited for massive deployments