# Hypervisor Architectures
## Type-1 and Type-2 Hypervisors in Detail

---

## Table of Contents
1. [VMware ESXi](#vmware-esxi)
2. [Xen Hypervisor](#xen-hypervisor)
3. [KVM (Kernel-based Virtual Machine)](#kvm)
4. [Microsoft Hyper-V](#microsoft-hyper-v)
5. [Type-2 Hypervisors](#type-2-hypervisors)
6. [Comparison](#comparison)

---

## Type-1 Hypervisors (Bare-Metal)

### VMware ESXi

**Architecture Overview:**

```
┌─────────────────────────────────────────┐
│  vSphere Client / API                   │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│       VMkernel (Core Hypervisor)        │
│  ┌───────────────────────────────────┐  │
│  │ VM Monitor (per VM component)     │  │
│  │   - CPU Scheduler                 │  │
│  │   - Memory Manager                │  │
│  │   - Device Emulation              │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ Device Drivers (vmklinux)         │  │
│  │   - Native VMware drivers         │  │
│  │   - Linux driver compatibility    │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**Key Components:**

1. **VMkernel** (~150MB footprint)
   - Microkernel architecture
   - CPU scheduling with proportional share
   - Memory management with page sharing
   - Direct device driver integration

2. **VM Monitor (VMM)**
   - One per VM, runs in Ring 0
   - Virtualizes CPU instructions
   - Handles privileged operations
   - Memory virtualization layer

3. **vmklinux**
   - Linux driver compatibility layer
   - Not a full Linux kernel
   - Enables use of Linux device drivers

**Memory Management Features:**
- **Transparent Page Sharing (TPS)**: Deduplicates identical pages
- **Memory Ballooning**: Dynamic memory reclamation
- **Memory Compression**: Compresses inactive pages
- **Swap to SSD**: Fast swap for overcommitment

---

### Xen Hypervisor

**Architecture Overview:**

```
┌────────────────────────────────────────────┐
│  Domain U (Unprivileged Guest VMs)         │
│  ┌──────┐  ┌──────┐  ┌──────┐              │
│  │ VM 1 │  │ VM 2 │  │ VM 3 │  HVM/PVH     │
│  └──────┘  └──────┘  └──────┘              │
├────────────────────────────────────────────┤
│  Domain 0 (Privileged Management Domain)   │
│  ┌────────────────────────────────────────┐│
│  │ Toolstack (xl/xm)                      ││
│  │ Backend Drivers                        ││
│  │ Device Models (QEMU)                   ││
│  └────────────────────────────────────────┘│
├────────────────────────────────────────────┤
│         Xen Hypervisor (~1MB)              │
│  │ Scheduler │ MMU │ Timer │ Event Ch. │   │
├────────────────────────────────────────────┤
│         Physical Hardware                  │
└────────────────────────────────────────────┘
```

**Key Components:**

1. **Xen Hypervisor (VMM)** - ~1MB footprint
   - Minimal microkernel design
   - CPU and memory virtualization
   - Event channels for communication
   - Grant tables for shared memory

2. **Domain 0 (Dom0)**
   - Privileged management domain
   - Runs full Linux kernel
   - All device drivers reside here
   - Toolstack for VM management
   - Backend drivers for I/O

3. **Domain U (DomU)**
   - Unprivileged guest domains
   - Multiple operating modes

**Virtualization Modes:**

| Mode | Description | Guest Modification |
|------|-------------|-------------------|
| **PV** | Paravirtualized, uses hypercalls | Yes (modified kernel) |
| **HVM** | Hardware Virtual Machine | No (unmodified) |
| **PVH** | Hybrid: HW for CPU/memory, PV for I/O | Minimal |

**Split Driver Model:**
```
DomU (Frontend)              Dom0 (Backend)
     │                            │
 Frontend Driver             Backend Driver
     │                            │
     └────Event Channel───────────┘
     └────Grant Table─────────────┘
           (shared memory)
```

**Communication Mechanisms:**
- **Event Channels**: Virtual interrupt mechanism
- **Grant Tables**: Explicit memory sharing
- **XenStore**: Configuration key-value database

---

### KVM (Kernel-based Virtual Machine)

**Architecture Overview:**

```
┌─────────────────────────────────────────────┐
│  User Space                                 │
│  ┌──────────────────────────────────────┐   │
│  │  QEMU Process (per VM)               │   │
│  │  ├─ Device Emulation                 │   │
│  │  ├─ BIOS (SeaBIOS/OVMF)              │   │
│  │  └─ VM Configuration                 │   │
│  └──────────────┬───────────────────────┘   │
│                 │ ioctl(/dev/kvm)           │
├─────────────────┼───────────────────────────┤
│  Kernel Space   │                           │
│  ┌──────────────▼───────────────────────┐   │
│  │  KVM Kernel Module                   │   │
│  │  ├─ /dev/kvm interface               │   │
│  │  ├─ vCPU management                  │   │
│  │  ├─ MMU virtualization (EPT/NPT)     │   │
│  │  └─ Interrupt handling               │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │  Linux Kernel                        │   │
│  │  ├─ Process Scheduler (CFS)          │   │
│  │  ├─ Memory Management                │   │
│  │  ├─ Device Drivers                   │   │
│  │  └─ File Systems                     │   │
│  └──────────────────────────────────────┘   │
├─────────────────────────────────────────────┤
│         Physical Hardware                   │
└─────────────────────────────────────────────┘
```

**Unique Hybrid Design:**
- Converts Linux kernel into Type-1 hypervisor
- Leverages existing kernel infrastructure
- Each vCPU is a Linux thread
- Uses Linux scheduler, memory manager

**Core Components:**

1. **KVM Kernel Module** (kvm.ko + kvm-intel.ko/kvm-amd.ko)
   - Provides `/dev/kvm` character device
   - Handles VM entry/exit
   - CPU virtualization via VT-x/AMD-V
   - Memory virtualization via EPT/NPT

2. **QEMU** (Quick Emulator)
   - Userspace component for each VM
   - Device emulation (chipset, PCI, USB, etc.)
   - BIOS/UEFI firmware loading
   - VirtIO device implementation
   - Management interface

3. **Linux Kernel Integration**
   - vCPUs scheduled by CFS
   - Memory managed by Linux MM
   - Access to full driver ecosystem
   - cgroups for resource control

**VM Execution Flow:**
```
1. QEMU creates VM via ioctl(/dev/kvm)
2. QEMU allocates memory for guest
3. QEMU creates vCPUs via ioctl
4. vCPU thread enters KVM module
5. KVM executes VMLAUNCH/VMRUN
6. Guest runs in VMX non-root mode
7. Privileged operation → VM Exit
8. KVM handles or passes to QEMU
9. VM Entry resumes guest
```

**VirtIO Framework:**
```
┌─────────────────────────────────────┐
│  Guest VM                           │
│  ┌───────────────────────────────┐  │
│  │  VirtIO Frontend Driver       │  │
│  │  (virtio-net, virtio-blk)     │  │
│  └───────────┬───────────────────┘  │
│              │ VirtQueue            │
├──────────────┼──────────────────────┤
│  QEMU        │                      │
│  ┌───────────▼───────────────────┐  │
│  │  VirtIO Backend               │  │
│  │  ├─ Process VirtQueue         │  │
│  │  └─ Access host resources     │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

---

### Microsoft Hyper-V

**Architecture Overview:**

```
┌──────────────────────────────────────────────┐
│  Virtual Machines (Child Partitions)         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │  VM 1   │  │  VM 2   │  │  VM 3   │       │
│  │  ├─VSC  │  │  ├─VSC  │  │  ├─VSC  │       │
│  └─────────┘  └─────────┘  └─────────┘       │
├──────────────────────────────────────────────┤
│  Parent Partition (Root)                     │
│  ┌────────────────────────────────────────┐  │
│  │  Windows Server                        │  │
│  │  ├─ Hyper-V Manager                    │  │
│  │  ├─ VSP (Virtualization Service        │  │
│  │  │        Providers)                   │  │
│  │  └─ WMI / PowerShell Management        │  │
│  └────────────────────────────────────────┘  │
├──────────────────────────────────────────────┤
│           Hyper-V Hypervisor                 │
│  │ Partitions │ Scheduler │ MMU │ IOMMU │    │
├──────────────────────────────────────────────┤
│         Physical Hardware                    │
└──────────────────────────────────────────────┘
```

**Key Concepts:**

1. **Partitions**
   - **Parent/Root Partition**: Privileged, runs full Windows
   - **Child Partitions**: Guest VMs with isolated address space

2. **Hypercalls**
   - Direct calls to hypervisor (more efficient than VM exits)
   - Used by enlightened guests

3. **Synthetic Devices**
   - **VMBus**: High-speed communication channel
   - **VSC** (Client) in guest ↔ **VSP** (Provider) in root

**Generation 1 vs Generation 2 VMs:**

| Feature | Gen 1 | Gen 2 |
|---------|-------|-------|
| Firmware | Legacy BIOS | UEFI |
| Storage | Emulated IDE | Synthetic SCSI |
| Architecture | 32/64-bit | 64-bit only |
| Secure Boot | No | Yes |
| Performance | Good | Better |

---

## Type-2 Hypervisors (Hosted)

### VMware Workstation/Fusion

**Components:**
- **vmmon**: Kernel module for VMM support
- **vmnet**: Virtual networking (bridged, NAT, host-only)
- **VMX Process**: Userspace VM executor (one per VM)

**Features:**
- Snapshots and cloning
- Unity mode (seamless windows)
- 3D graphics acceleration

### Oracle VirtualBox

**Components:**
- **VBoxDrv**: Kernel module for hardware virtualization
- **VBoxSVC**: Background service for VM lifecycle
- **VBoxVMM**: Core virtualization engine

**Execution Modes:**
- Hardware Virtualization (VT-x/AMD-V)
- Software Virtualization/Recompiler
- Paravirtualization interface

### QEMU (Without KVM)

**Pure Emulation Mode:**
- Uses TCG (Tiny Code Generator) for binary translation
- JIT compilation of guest instructions
- 10-100x slower than native
- Useful for cross-architecture development

---

## Comparison Matrix

### Type-1 Hypervisor Comparison

| Feature | VMware ESXi | Xen | KVM | Hyper-V |
|---------|-------------|-----|-----|---------|
| Architecture | Microkernel | Microkernel | Linux-integrated | Microkernel |
| Footprint | ~150MB | ~1MB | Kernel module | ~20MB |
| Management | vCenter | xl/XAPI | libvirt/virsh | SCVMM |
| Live Migration | vMotion | Xen Motion | Libvirt | Live Migration |
| Overhead | Very Low | Very Low | Low | Low |
| Ecosystem | Largest | Cloud-focused | Linux | Microsoft stack |

### Architecture Comparison Matrix

| Type | Isolation | Performance | Startup | Memory | Use Case |
|------|-----------|-------------|---------|--------|----------|
| Type-1 Hypervisor | Strong (HW) | 95%+ | Minutes | GB/VM | Production |
| Type-2 Hypervisor | Strong (HW) | 80-90% | Minutes | GB/VM | Desktop/Dev |
| Containers | Moderate (NS) | 99%+ | Seconds | MB | Microservices |
| Kata Containers | Strong (HW) | 90-95% | ~100ms | ~120MB | Secure multi-tenant |
| gVisor | Good | 70-90% | <1s | ~50MB | Untrusted code |
| Firecracker | Strong (HW) | 95%+ | <125ms | ~5MB | Serverless/FaaS |

---

*Previous: [02_Hardware_Based_Virtualization.md](02_Hardware_Based_Virtualization.md)*
*Next: [04_Linux_Kernel_Virtualization_Patches.md](04_Linux_Kernel_Virtualization_Patches.md)*
