## Core KVM (Kernel-based Virtual Machine) Patches

**Initial KVM Integration (2007)**
- Merged in kernel 2.6.20
- Introduced x86 virtualization support using Intel VT-x and AMD-V
- Created `/dev/kvm` interface for userspace hypervisors

**KVM ARM Support (2013)**
- Kernel 3.9 introduced KVM for ARM architectures
- ARM64 support added in kernel 3.11
- Enabled virtualization on ARM-based servers and mobile devices

**Nested Virtualization**
- Intel nested VMX support (kernel 3.1, 2011)
- AMD nested SVM support (kernel 3.3, 2012)
- Allows running hypervisors inside VMs

## Memory Management for Virtualization

**EPT/NPT Support (Extended Page Tables/Nested Page Tables)**
- Kernel 2.6.26 (2008)
- Hardware-assisted memory virtualization
- Dramatically improved VM performance by eliminating shadow page tables

**KSM (Kernel Samepage Merging)**
- Kernel 2.6.32 (2009)
- Deduplicates identical memory pages across VMs
- Significant memory savings in virtualized environments

**Transparent Huge Pages (THP)**
- Kernel 2.6.38 (2011)
- Automatic huge page allocation for VMs
- Improved memory performance without manual configuration

**Memory Ballooning**
- virtio-balloon driver
- Dynamic memory reclamation from VMs
- Better host memory utilization

## CPU and Performance Features

**VPID/ASID Support**
- Virtual Processor ID (Intel) and Address Space ID (AMD)
- Kernel 2.6.26
- Reduced TLB flush overhead during VM context switches

**Posted Interrupts**
- Kernel 3.17 (2014)
- Direct interrupt delivery to VMs without VM exits
- Reduced virtualization overhead

**CPU Hotplug for VMs**
- Dynamic vCPU addition/removal
- Various patches across kernels 3.x series

**Para-virtualized Spinlocks**
- Kernel 3.8 (2013)
- Optimized lock handling in virtualized environments
- Reduced CPU wastage from spinlock contention

## I/O Virtualization

**virtio Framework**
- Kernel 2.6.24 (2008)
- Standardized I/O virtualization interface
- virtio-net, virtio-blk, virtio-scsi, virtio-console, virtio-balloon

**VFIO (Virtual Function I/O)**
- Kernel 3.6 (2012)
- Safe, direct device assignment to VMs
- IOMMU-based device isolation
- Foundation for GPU and high-performance device passthrough

**SR-IOV Support**
- Single Root I/O Virtualization
- Multiple kernels, stabilized around 3.x
- Hardware-level device virtualization

**vhost and vhost-net**
- Kernel 2.6.35 (2010)
- In-kernel virtio backend for better performance
- vhost-scsi added in kernel 3.6
- vhost-vsock for VM-to-VM communication (kernel 4.8)

**VDPA (vDPA - Virtio Data Path Acceleration)**
- Kernel 5.7 (2020)
- Hardware offload for virtio devices
- Bridges virtio and hardware acceleration

## Container and Namespace Technologies

**Namespaces** (foundation for containers)
- Mount namespaces: kernel 2.4.19 (2002)
- UTS namespaces: kernel 2.6.19 (2006)
- IPC namespaces: kernel 2.6.19 (2006)
- PID namespaces: kernel 2.6.24 (2008)
- Network namespaces: kernel 2.6.29 (2009)
- User namespaces: kernel 3.8 (2013)
- Cgroup namespaces: kernel 4.6 (2016)
- Time namespaces: kernel 5.6 (2020)

**cgroups (Control Groups)**
- v1 initial merge: kernel 2.6.24 (2008)
- cgroups v2: kernel 4.5 (2016)
- Memory controller, CPU controller, blkio controller
- Essential for Docker, Kubernetes, and container resource management

**Overlay Filesystem**
- Kernel 3.18 (2014)
- Union mount filesystem
- Critical for container image layers

**seccomp (Secure Computing Mode)**
- Basic seccomp: kernel 2.6.12 (2005)
- seccomp-bpf: kernel 3.5 (2012)
- Container syscall filtering

## Hypervisor Support and Paravirtualization

**Xen Support**
- Dom0 support merged in kernel 2.6.37 (2011)
- Xen frontend/backend drivers throughout 2.6.x series
- PVH mode support (kernel 4.11, 2017)

**Hyper-V Support**
- Initial Linux Integration Services: kernel 2.6.32 (2009)
- VMBus, synthetic network/storage devices
- Hyper-V enlightenments for KVM guests (kernel 4.13+)

**VMware Support**
- vmw_balloon driver
- vmxnet3 network driver
- VMCI (Virtual Machine Communication Interface)

## Security Features for Virtualization

**SEV (Secure Encrypted Virtualization)**
- AMD SEV support: kernel 4.16 (2018)
- SEV-ES: kernel 5.10 (2020)
- SEV-SNP patches ongoing
- Memory encryption for VM isolation

**Intel TDX (Trust Domain Extensions)**
- Initial support: kernel 5.19 (2022)
- Confidential computing for VMs

**Memory Encryption**
- AMD SME/SEV framework
- Intel TME support

## Device and Driver Virtualization

**GPU Virtualization**
- GVT-g (Intel Graphics Virtualization): kernel 4.10 (2017)
- VFIO-mdev framework: kernel 4.10 (2017)
- Mediated device framework for GPU sharing

**IOMMU Improvements**
- Intel VT-d support: kernel 2.6.26
- AMD IOMMU support: kernel 2.6.26
- IOMMU groups and isolation
- Scalable mode support (kernel 5.0+)

## Network Virtualization

**macvlan/macvtap**
- kernel 2.6.33 (2010)
- Network virtualization without bridges

**veth (Virtual Ethernet)**
- Container networking primitive
- Kernel 2.6.23 (2007)

**eBPF/XDP for Virtualization**
- eBPF infrastructure: kernel 3.18+
- XDP: kernel 4.8 (2016)
- High-performance packet processing for containers/VMs

**Network Namespaces Enhancements**
- VRF (Virtual Routing and Forwarding): kernel 4.3 (2015)
- Better network isolation

## Live Migration and Checkpointing

**CRIU Support (Checkpoint/Restore in Userspace)**
- Kernel support patches across 3.x series
- Process and container checkpointing

**Post-copy Migration**
- Userfaultfd: kernel 4.3 (2015)
- Enables efficient live migration

## Recent Virtualization Advances

**io_uring for VMs**
- Kernel 5.1 (2019)
- High-performance async I/O
- Benefits virtualized storage

**KVM Scalability**
- Async page fault handling
- Large page support improvements
- Multi-queue virtio improvements

**Real-time Virtualization**
- PREEMPT_RT integration
- Low-latency VM support

**Rust in Kernel (ongoing)**
- Kernel 6.1+ (2022+)
- Will impact future virtualization drivers

## Cloud-Native and Container-Specific

**Cgroupfs v2 Improvements**
- PSI (Pressure Stall Information): kernel 4.20
- Better resource accounting

**pidfd**
- Kernel 5.3 (2019)
- Race-free process management for containers

**Landlock LSM**
- Kernel 5.13 (2021)
- Sandboxing for unprivileged processes

## Architecture-Specific

**ARM Virtualization**
- KVM/ARM: kernel 3.9
- ARM64 nested virtualization: kernel 5.16 (2022)

**RISC-V Virtualization**
- KVM/RISC-V: kernel 5.17 (2022)

**IBM Z/s390x**
- KVM for s390: kernel 3.16 (2014)

This represents the major virtualization-related patches in the Linux kernel. The virtualization ecosystem continues to evolve with ongoing improvements in security (confidential computing), performance (hardware offload), and functionality (new device types, better isolation).