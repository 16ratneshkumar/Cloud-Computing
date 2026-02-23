[← Back to Index](../index.md)

# DEPLOYMENT 7 — Bare Metal → KVM → QEMU VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                    QEMU VMs                    │
├──────────────────────────────────────────────────┤
│                      KVM                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                  GUEST VIRTUAL MACHINES                         │
│  ┌────────────────────┐  ┌────────────────────┐  ┌──────────┐  │
│  │    Guest VM A      │  │    Guest VM B      │  │  Guest C │  │
│  │  Windows Server    │  │  Ubuntu Linux      │  │  CentOS  │  │
│  │  virtio-net (NIC)  │  │  virtio-net (NIC)  │  │  virtio  │  │
│  │  virtio-blk (disk) │  │  virtio-blk (disk) │  │          │  │
│  │  vCPU 1-4          │  │  vCPU 1-8          │  │  vCPU    │  │
│  │  RAM: 8GB          │  │  RAM: 16GB         │  │  RAM:4GB │  │
│  └────────────────────┘  └────────────────────┘  └──────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │  each VM is a QEMU process
┌────────────────────────────▼────────────────────────────────────┐
│               QEMU (Machine Emulator + Virtualizer)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  One QEMU process per VM                                 │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐ │   │
│  │  │ virtio-net   │  │ virtio-blk   │  │  SPICE/VNC     │ │   │
│  │  │ (tap device) │  │ (image file/ │  │  (VM console)  │ │   │
│  │  │              │  │  block dev)  │  │                │ │   │
│  │  └──────────────┘  └──────────────┘  └────────────────┘ │   │
│  │  ┌──────────────────────────────────────────────────────┐│   │
│  │  │  QEMU monitor (QMP socket) — hot-plug, migration    ││   │
│  │  └──────────────────────────────────────────────────────┘│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  ioctls to /dev/kvm
┌────────────────────────────▼────────────────────────────────────┐
│             KVM (Kernel Virtual Machine)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kvm.ko + kvm-intel.ko / kvm-amd.ko  (kernel modules)   │   │
│  │  /dev/kvm  (character device — QEMU talks to this)       │   │
│  │  ┌──────────────┐  ┌────────────────┐  ┌──────────────┐ │   │
│  │  │  VMX/SVM     │  │  EPT / NPT     │  │  VPID / ASID │ │   │
│  │  │  (CPU virt)  │  │  (mem virt)    │  │  (TLB tags)  │ │   │
│  │  └──────────────┘  └────────────────┘  └──────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (HOST)                          │
│  ┌────────────┐  ┌────────────────┐  ┌────────────────────┐    │
│  │  virtio    │  │  TUN/TAP       │  │  iptables/         │    │
│  │  (fast I/O │  │  (VM network   │  │  nftables          │    │
│  │   protocol)│  │   tap devices) │  │  (VM firewall)     │    │
│  └────────────┘  └────────────────┘  └────────────────────┘    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  ┌───────────────┐  ┌───────────────┐  ┌────────────────────┐  │
│  │  CPU          │  │  RAM          │  │  Storage           │  │
│  │  Intel VT-x   │  │  (split by    │  │  (qcow2 / raw /   │  │
│  │  or AMD-V     │  │   VMs)        │  │   LVM / iSCSI)    │  │
│  └───────────────┘  └───────────────┘  └────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  NIC (physical) → Linux bridge (virbr0) → VM tap devs    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- KVM kernel module (kvm.ko + kvm-intel/amd.ko) exposes /dev/kvm character device
- QEMU opens /dev/kvm and uses ioctls to create a virtual CPU (vCPU) using hardware virtualization (Intel VT-x / AMD-V)
- Extended Page Tables (EPT/NPT) handle guest memory translation in hardware
- Each VM is a QEMU process on the host with virtio devices for fast I/O
- Linux TUN/TAP provides virtual network interfaces for VMs connected to Linux bridges

## Use Case
Foundation of almost all Linux-based virtualization. Standalone VM hosting, development VMs, foundation layer for Proxmox, OpenStack, oVirt, and more.

## Pros
- Industry-standard open-source hypervisor
- Near-native performance with VT-x/AMD-V and EPT
- Supports Windows, Linux, BSD, and others as guests
- Full ecosystem: libvirt, Proxmox, OpenStack built on top
- Active upstream — part of Linux kernel

## Cons
- Raw KVM/QEMU requires manual setup (no GUI)
- Networking setup complex without libvirt
- No built-in clustering or migration without tools
- Requires hardware virtualization support