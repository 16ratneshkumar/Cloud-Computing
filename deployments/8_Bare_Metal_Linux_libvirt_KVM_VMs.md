[← Back to Index](../index.md)

# DEPLOYMENT 8 — Bare Metal → Linux → libvirt → KVM VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                    KVM VMs                     │
├──────────────────────────────────────────────────┤
│                    libvirt                     │
├──────────────────────────────────────────────────┤
│                     Linux                      │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT INTERFACES                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  virsh (CLI) │  │ virt-manager │  │  cockpit-machines    │  │
│  │  virsh list  │  │ (GTK GUI)    │  │  (web UI)            │  │
│  │  virsh start │  │              │  │                      │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
└─────────┼─────────────────┼────────────────────  ┼─────────────┘
          └─────────────────┼─────────────────────-┘
                            │  libvirt API (Unix socket / TLS)
┌───────────────────────────▼─────────────────────────────────────┐
│                   libvirt (libvirtd / virtqemud)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Domain XML Management  (VM definitions in /etc/libvirt/) │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │ Storage Pools    │  │  Network Mgmt    │  │  Secret Mgmt │  │
│  │  ┌────────────┐  │  │  ┌───────────┐  │  │  (passwords/ │  │
│  │  │ dir pool   │  │  │  │ NAT net   │  │  │  TLS certs)  │  │
│  │  │ LVM pool   │  │  │  │ bridge net│  │  │              │  │
│  │  │ RBD pool   │  │  │  │ macvtap   │  │  │              │  │
│  │  │ iSCSI pool │  │  │  │ SR-IOV    │  │  │              │  │
│  │  └────────────┘  │  │  └───────────┘  │  │              │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Hypervisor Drivers: QEMU/KVM │ LXC │ OpenVZ │ VMware   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  launches QEMU processes
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU + KVM  (one process per VM)                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Guest VM A       │  Guest VM B       │  Guest VM C      │   │
│  │  [Windows Server] │  [Ubuntu 22.04]   │  [RHEL 9]        │   │
│  │  8 vCPU / 32GB    │  4 vCPU / 16GB   │  2 vCPU / 8GB    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                 KVM kernel module + LINUX KERNEL                 │
│       VT-x/AMD-V │ EPT/NPT │ TUN/TAP │ bridge │ iptables        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   CPU (VT-x/AMD-V) │ RAM │ Storage (LVM/qcow2/iSCSI) │ NIC     │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- libvirtd daemon provides a unified API over various hypervisors (mainly QEMU/KVM)
- virsh, virt-manager, and cockpit connect to libvirtd via Unix socket or TLS
- VM definitions stored as XML files in /etc/libvirt/qemu/
- libvirt manages storage pools (LVM, qcow2 files, Ceph RBD, iSCSI)
- libvirt creates virtual networks using TUN/TAP + Linux bridges for VM networking
- Also used by OpenStack Nova as the compute driver

## Use Case
Single-host or small-cluster KVM management. Foundation for OpenStack Nova, oVirt, and Proxmox. Standard tool for Linux administrators managing VMs.

## Pros
- Standard API for KVM management across many tools
- Rich storage and network management
- Supports multiple hypervisor backends
- Base of OpenStack, oVirt, Proxmox

## Cons
- Single-host by default (no built-in clustering)
- XML configuration can be verbose
- Live migration requires shared storage or additional setup
- No built-in HA