# Linux Kernel Virtualization Patches
## Historical Evolution of Virtualization in Linux

---

## Table of Contents
1. [Core KVM Patches](#core-kvm-patches)
2. [Memory Management](#memory-management)
3. [CPU and Performance](#cpu-and-performance)
4. [I/O Virtualization](#io-virtualization)
5. [Container Technologies](#container-technologies)
6. [Security Features](#security-features)

---

## Core KVM (Kernel-based Virtual Machine) Patches

### Initial KVM Integration (2007)
- **Kernel 2.6.20** - KVM merged into mainline Linux
- Introduced x86 virtualization using Intel VT-x and AMD-V
- Created `/dev/kvm` interface for userspace hypervisors
- Marked Linux's entry into the hypervisor space

### KVM ARM Support (2013)
- **Kernel 3.9** - KVM for ARM architectures
- **Kernel 3.11** - ARM64 (AArch64) support added
- Enabled virtualization on ARM-based servers and mobile devices
- Uses ARMv8 Exception Level 2 (EL2)

### Nested Virtualization
- **Kernel 3.1 (2011)** - Intel nested VMX support
- **Kernel 3.3 (2012)** - AMD nested SVM support
- Allows running hypervisors inside VMs
- Use case: Testing hypervisors, KVM-on-KVM

---

## Memory Management for Virtualization

### EPT/NPT Support (2008)
- **Kernel 2.6.26** - Extended Page Tables (Intel) / Nested Page Tables (AMD)
- Hardware-assisted memory virtualization
- Dramatically improved VM performance
- Eliminated need for shadow page tables

### KSM - Kernel Same-page Merging (2009)
- **Kernel 2.6.32**
- Deduplicates identical memory pages across VMs
- Scans pages, creates shared copy-on-write mappings
- Significant memory savings with similar VMs

### Transparent Huge Pages (2011)
- **Kernel 2.6.38**
- Automatic huge page (2MB/1GB) allocation for VMs
- Improved memory performance
- No manual configuration required

### Memory Ballooning
- `virtio-balloon` driver
- Dynamic memory reclamation from VMs
- Cooperative memory management between host and guest
- Better overall host memory utilization

---

## CPU and Performance Features

### VPID/ASID Support
- **Kernel 2.6.26**
- Virtual Processor ID (Intel) / Address Space ID (AMD)
- Reduced TLB flush overhead during VM context switches
- Each VM gets unique VPID, preserving TLB entries

### Posted Interrupts (2014)
- **Kernel 3.17**
- Direct interrupt delivery to VMs without VM exits
- Hardware posts interrupts directly to guest vCPU
- Reduced virtualization overhead for I/O-heavy workloads

### CPU Hotplug for VMs
- Dynamic vCPU addition/removal at runtime
- Implemented across kernel 3.x series
- Enables VM resource scaling

### Para-virtualized Spinlocks (2013)
- **Kernel 3.8**
- Optimized lock handling in virtualized environments
- Reduces CPU wastage from spinlock contention
- Uses hypercalls to yield CPU when lock unavailable

---

## I/O Virtualization

### VirtIO Framework (2008)
- **Kernel 2.6.24**
- Standardized I/O virtualization interface
- Components:
  - `virtio-net` - Network devices
  - `virtio-blk` - Block storage
  - `virtio-scsi` - SCSI controller
  - `virtio-console` - Console device
  - `virtio-balloon` - Memory ballooning

### VFIO (Virtual Function I/O) - 2012
- **Kernel 3.6**
- Safe, direct device assignment to VMs
- IOMMU-based device isolation
- Foundation for GPU and high-performance device passthrough
- Replaced older UIO approach

### SR-IOV Support
- **Multiple kernels, stabilized around 3.x**
- Single Root I/O Virtualization
- Hardware-level device virtualization
- Physical device presents multiple virtual functions

### vhost and vhost-net (2010)
- **Kernel 2.6.35** - vhost-net
- **Kernel 3.6** - vhost-scsi
- **Kernel 4.8** - vhost-vsock (VM-to-VM communication)
- In-kernel VirtIO backend for better performance
- Eliminates QEMU context switches for I/O

### VDPA (vDPA - Virtio Data Path Acceleration) - 2020
- **Kernel 5.7**
- Hardware offload for VirtIO devices
- Bridges VirtIO interface with hardware acceleration
- Enables SR-IOV NICs to present VirtIO interface

---

## Container and Namespace Technologies

### Namespaces (Foundation for Containers)

| Namespace | Kernel Version | Purpose |
|-----------|---------------|---------|
| Mount | 2.4.19 (2002) | Filesystem isolation |
| UTS | 2.6.19 (2006) | Hostname/domain isolation |
| IPC | 2.6.19 (2006) | Inter-process communication |
| PID | 2.6.24 (2008) | Process ID isolation |
| Network | 2.6.29 (2009) | Network stack isolation |
| User | 3.8 (2013) | UID/GID mapping |
| Cgroup | 4.6 (2016) | Control group visibility |
| Time | 5.6 (2020) | Time namespace |

### cgroups (Control Groups)
- **Kernel 2.6.24 (2008)** - cgroups v1 initial merge
- **Kernel 4.5 (2016)** - cgroups v2 (unified hierarchy)
- Controllers: Memory, CPU, blkio, devices, freezer, pids
- Essential for Docker, Kubernetes, container resource management

### Overlay Filesystem (2014)
- **Kernel 3.18**
- Union mount filesystem
- Critical for container image layers
- Enables copy-on-write for containers

### seccomp (Secure Computing Mode)
- **Kernel 2.6.12 (2005)** - Basic seccomp
- **Kernel 3.5 (2012)** - seccomp-bpf with filtering
- Container syscall filtering
- Reduces kernel attack surface

---

## Hypervisor Support and Paravirtualization

### Xen Support
- **Kernel 2.6.37 (2011)** - Dom0 support merged
- Xen frontend/backend drivers throughout 2.6.x series
- **Kernel 4.11 (2017)** - PVH mode support

### Hyper-V Support
- **Kernel 2.6.32 (2009)** - Initial Linux Integration Services
- VMBus, synthetic network/storage devices
- **Kernel 4.13+** - Hyper-V enlightenments for KVM guests

### VMware Support
- vmw_balloon driver - Memory ballooning
- vmxnet3 - High-performance network driver
- VMCI - Virtual Machine Communication Interface

---

## Security Features for Virtualization

### AMD SEV (2018)
- **Kernel 4.16** - AMD SEV (Secure Encrypted Virtualization)
- **Kernel 5.10 (2020)** - SEV-ES (Encrypted State)
- SEV-SNP patches ongoing
- Memory encryption for VM isolation

### Intel TDX (2022)
- **Kernel 5.19** - Initial Trust Domain Extensions support
- Confidential computing for VMs
- Hardware-isolated trust domains

### Memory Encryption
- AMD SME/SEV framework
- Intel TME support
- Protects VM memory even from hypervisor

---

## Device and Driver Virtualization

### GPU Virtualization (2017)
- **Kernel 4.10** - Intel GVT-g (Graphics Virtualization)
- **Kernel 4.10** - VFIO-mdev framework
- Mediated device framework for GPU sharing
- Enables GPU passthrough and time-slicing

### IOMMU Improvements
- **Kernel 2.6.26** - Intel VT-d support
- **Kernel 2.6.26** - AMD IOMMU support
- IOMMU groups and device isolation
- **Kernel 5.0+** - Scalable mode support

---

## Network Virtualization

### macvlan/macvtap (2010)
- **Kernel 2.6.33**
- Network virtualization without bridges
- Each container gets virtual MAC address

### veth (Virtual Ethernet) - 2007
- **Kernel 2.6.23**
- Container networking primitive
- Creates paired virtual interfaces

### eBPF/XDP for Virtualization
- **Kernel 3.18+** - eBPF infrastructure
- **Kernel 4.8 (2016)** - XDP (eXpress Data Path)
- High-performance packet processing
- Programmable network functions

### VRF (Virtual Routing and Forwarding) - 2015
- **Kernel 4.3**
- Network namespace isolation enhancement
- Layer 3 traffic separation

---

## Live Migration and Checkpointing

### CRIU Support (Checkpoint/Restore in Userspace)
- Kernel support patches across 3.x series
- Process and container checkpointing
- Enables container live migration

### Post-copy Migration (2015)
- **Kernel 4.3** - Userfaultfd
- Enables efficient live migration
- Pages transferred on-demand after migration

---

## Recent Virtualization Advances

### io_uring for VMs (2019)
- **Kernel 5.1**
- High-performance async I/O
- Benefits virtualized storage significantly
- Reduces syscall overhead

### KVM Scalability
- Async page fault handling
- Large page support improvements
- Multi-queue VirtIO enhancements

### Real-time Virtualization
- PREEMPT_RT integration
- Low-latency VM support
- Deterministic scheduling

### Rust in Kernel (2022+)
- **Kernel 6.1+**
- Memory-safe driver development
- Will impact future virtualization drivers

---

## Cloud-Native and Container-Specific

### Cgroupfs v2 Improvements
- **Kernel 4.20** - PSI (Pressure Stall Information)
- Better resource accounting
- Unified hierarchy for all controllers

### pidfd (2019)
- **Kernel 5.3**
- Race-free process management for containers
- File descriptor-based process handle

### Landlock LSM (2021)
- **Kernel 5.13**
- Sandboxing for unprivileged processes
- Complementary to seccomp

---

## Architecture-Specific Virtualization

### ARM Virtualization
- **Kernel 3.9** - KVM/ARM
- **Kernel 5.16 (2022)** - ARM64 nested virtualization

### RISC-V Virtualization
- **Kernel 5.17 (2022)** - KVM/RISC-V

### IBM Z/s390x
- **Kernel 3.16 (2014)** - KVM for s390

---

## Summary Timeline

```
2002: Mount namespaces (first isolation primitive)
2007: KVM merged (2.6.20), veth added
2008: cgroups v1, PID/IPC namespaces, VirtIO, EPT/NPT
2009: KSM, Network namespaces, Hyper-V integration
2010: vhost-net, macvlan
2011: THP, Xen Dom0 support
2012: VFIO, seccomp-bpf
2013: User namespaces, KVM/ARM, PV spinlocks
2014: Overlay FS, Posted Interrupts
2016: cgroups v2, XDP
2017: Intel GVT-g, VFIO-mdev
2018: AMD SEV
2019: io_uring, pidfd
2020: Time namespaces, VDPA, SEV-ES
2021: Landlock LSM
2022: Intel TDX, ARM64 nested virt, RISC-V KVM
```

---

*Previous: [03_Hypervisor_Architectures.md](03_Hypervisor_Architectures.md)*
*Next: [05_Container_Technologies_and_cgroups.md](05_Container_Technologies_and_cgroups.md)*
