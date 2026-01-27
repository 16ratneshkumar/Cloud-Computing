# KVM in Cloud Providers and Libvirt
## Cloud Infrastructure and Virtualization Management

---

## Table of Contents
1. [AWS Nitro System](#aws-nitro-system)
2. [Google Cloud Platform](#google-cloud-platform)
3. [Microsoft Azure](#microsoft-azure)
4. [Libvirt Management](#libvirt-management)

---

## AWS Nitro System

### Evolution from Xen to KVM

```
Timeline:
2006-2010: Xen (paravirtualization)
2011-2017: Xen HVM (hardware virtualization)
2017-Present: Nitro System (KVM-based)
```

### Nitro System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    EC2 Instance                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Guest OS (Linux/Windows/etc.)                     │ │
│  │  Customer Applications                             │ │
│  └────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────┤
│              Nitro Hypervisor (KVM-based)               │
│  ├─ Lightweight KVM                                     │
│  ├─ CPU/Memory virtualization ONLY                      │
│  └─ Minimal attack surface                              │
├─────────────────────────────────────────────────────────┤
│              Nitro Cards (Custom ASICs)                 │
│  ┌──────────────┬──────────────┬──────────────────────┐ │
│  │ Nitro VPC    │ Nitro EBS    │ Nitro Security       │ │
│  │ (Networking) │ (Storage)    │ (Encryption, TPM)    │ │
│  │ Up to 100G   │ Up to 64K    │ Hardware Root of     │ │
│  │ throughput   │ IOPS         │ Trust                │ │
│  └──────────────┴──────────────┴──────────────────────┘ │
├─────────────────────────────────────────────────────────┤
│              Physical Hardware                          │
│  Custom AWS Server Hardware (Intel/AMD/Graviton)        │
└─────────────────────────────────────────────────────────┘
```

### Nitro Components

**Nitro Hypervisor:**
- Based on core KVM from Linux kernel
- ONLY handles CPU and memory virtualization
- NO device emulation (offloaded to cards)
- Extremely minimal (~3-5% overhead)

**Nitro VPC Card:**
- Virtual Network Interface
- Security Groups (firewall rules)
- NAT and Routing
- Up to 100 Gbps throughput

**Nitro EBS Card:**
- NVMe controller
- Encryption/Decryption engine
- NVMe-over-fabric protocol
- Up to 64,000 IOPS

**Nitro Security Chip:**
- Hardware Root of Trust
- Secure Boot verification
- Memory encryption controller
- Prevents admin access to VMs

### Why AWS Chose KVM

1. **Minimal Footprint** - Reduces attack surface
2. **Hardware Offload** - All I/O to Nitro cards
3. **Linux Ecosystem** - Active development community
4. **Bare Metal Support** - `.metal` instances with minimal overhead

### AWS Firecracker (Lambda & Fargate)

```
┌────────────────────────────────────────┐
│  Lambda Function / Fargate Task        │
├────────────────────────────────────────┤
│  Firecracker MicroVM                   │
│  ├─ Custom KVM-based VMM (Rust)        │
│  ├─ <125ms cold start                  │
│  ├─ 5MB memory overhead                │
│  └─ Minimal device model               │
├────────────────────────────────────────┤
│  Host Linux (KVM enabled)              │
└────────────────────────────────────────┘
```

**Characteristics:**
- Written in Rust (memory-safe)
- Only virtio-net and virtio-block
- Ultra-secure for multi-tenancy
- Powers serverless at scale

---

## Google Cloud Platform

### GCP Virtualization Stack

```
┌─────────────────────────────────────────────┐
│         Compute Engine VM Instance          │
│  ┌───────────────────────────────────────┐  │
│  │  Guest OS (Linux/Windows)             │  │
│  │  Customer Applications                │  │
│  └───────────────────────────────────────┘  │
├─────────────────────────────────────────────┤
│         Google's Custom KVM Hypervisor      │
│  ├─ Modified KVM (Google optimizations)     │
│  ├─ VirtIO drivers                          │
│  └─ Live migration support                  │
├─────────────────────────────────────────────┤
│         Andromeda (SDN Stack)               │
│  ├─ Software-defined networking             │
│  └─ Flow-based routing                      │
├─────────────────────────────────────────────┤
│         Colossus (Distributed Storage)      │
│  └─ Persistent Disks                        │
├─────────────────────────────────────────────┤
│         Physical Hardware                   │
└─────────────────────────────────────────────┘
```

### Google's KVM Innovations

**Live Migration at Scale:**
```
Source Host                 Destination Host
     │                            │
     ├── Pre-copy memory ────────>│
     │   (iterative)              │
     ├── Dirty page tracking ────>│
     │                            │
     ├── Final sync ─────────────>│
     │   (<1 second blackout)     │
     └── VM continues on dest ───>│
```

- Post-copy live migration
- Less than 1 second downtime
- Transparent to guest
- Used for hardware maintenance

**gVisor (Container Isolation):**
```
┌────────────────────────────────────┐
│  Container Application             │
├────────────────────────────────────┤
│  Sentry (User-space Kernel)        │
│  ├─ Implements Linux syscalls      │
│  └─ Written in Go                  │
├────────────────────────────────────┤
│  KVM Platform (optional)           │
│  ├─ Hardware isolation             │
│  └─ Uses KVM for syscall trap      │
├────────────────────────────────────┤
│  Host Kernel                       │
└────────────────────────────────────┘
```

---

## Microsoft Azure

### Hybrid Approach

Azure primarily uses **Hyper-V**, with KVM for specific workloads:

**Main Stack (Hyper-V):**
```
┌────────────────────────────────────┐
│  Azure VM                          │
├────────────────────────────────────┤
│  Hyper-V Hypervisor                │
│  ├─ Windows Hypervisor Platform    │
│  └─ VMBus for I/O                  │
├────────────────────────────────────┤
│  Azure Host OS (Windows)           │
└────────────────────────────────────┘
```

**Linux Workloads (KVM with Kata):**
```
┌────────────────────────────────────┐
│  Container Application             │
├────────────────────────────────────┤
│  Kata Container Runtime            │
│  ├─ Lightweight VM (KVM)           │
│  └─ Fast startup (~100ms)          │
├────────────────────────────────────┤
│  Azure Host (CBL-Mariner Linux)    │
└────────────────────────────────────┘
```

**CBL-Mariner:** Microsoft's own Linux distribution for internal infrastructure.

---

## Libvirt Management

### What is Libvirt?

Libvirt is a toolkit/API providing a unified interface across different hypervisors.

```
┌──────────────────────────────────────────────────┐
│           Management Tools                       │
│  virsh │ virt-manager │ OpenStack │ oVirt        │
├──────────────────────────────────────────────────┤
│                 libvirt API                      │
│  ├─ VM lifecycle                                 │
│  ├─ Storage management                           │
│  ├─ Network management                           │
│  └─ Live migration                               │
├──────────────────────────────────────────────────┤
│            libvirt Drivers                       │
│  KVM │ Xen │ QEMU │ LXC │ VirtualBox             │
├──────────────────────────────────────────────────┤
│            Hypervisors                           │
└──────────────────────────────────────────────────┘
```

### Connection URIs

```bash
# Local QEMU/KVM (most common)
qemu:///system      # System-wide (root required)
qemu:///session     # User session

# Remote connections
qemu+ssh://user@host/system
qemu+tcp://host/system

# Other hypervisors
xen:///             # Xen
lxc:///             # Linux Containers
vbox:///session     # VirtualBox
```

### Domain XML Configuration

```xml
<domain type='kvm'>
  <name>my-vm</name>
  <memory unit='GiB'>4</memory>
  <vcpu placement='static'>2</vcpu>
  
  <cpu mode='host-passthrough'>
    <topology sockets='1' cores='2' threads='1'/>
  </cpu>
  
  <os>
    <type arch='x86_64' machine='pc-q35-6.2'>hvm</type>
    <boot dev='hd'/>
  </os>
  
  <devices>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/my-vm.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
    </interface>
  </devices>
</domain>
```

### VM Startup Flow

```
virsh start my-vm
         ↓
libvirtd receives request
         ↓
QEMU driver loads domain XML
         ↓
Build QEMU command line:
  /usr/bin/qemu-system-x86_64 \
    -machine pc-q35-6.2,accel=kvm \
    -cpu host -m 4096 -smp 2 \
    -drive file=my-vm.qcow2,if=virtio \
    -enable-kvm
         ↓
Execute QEMU process
         ↓
QEMU opens /dev/kvm
         ↓
KVM creates VM in kernel
         ↓
VM running
```

### Common virsh Commands

```bash
# VM Management
virsh list --all                    # List all VMs
virsh start my-vm                   # Start VM
virsh shutdown my-vm                # Graceful shutdown
virsh destroy my-vm                 # Force stop

# VM Definition
virsh define my-vm.xml              # Define from XML
virsh dumpxml my-vm                 # Show VM XML
virsh edit my-vm                    # Edit config

# Snapshots
virsh snapshot-create-as my-vm snap1 "First snapshot"
virsh snapshot-list my-vm
virsh snapshot-revert my-vm snap1

# Live Migration
virsh migrate --live my-vm qemu+ssh://dest-host/system

# Network
virsh net-list --all
virsh net-start default

# Storage
virsh pool-list --all
virsh vol-create-as default vm5.qcow2 50G --format qcow2
```

### Libvirt Network Management

```
┌─────────────────────────────────────────┐
│  Virtual Network: "default"             │
│  Network: 192.168.122.0/24              │
│  DHCP: 192.168.122.2-254                │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  virbr0 (Linux Bridge)                  │
│  IP: 192.168.122.1                      │
└──┬─────────┬─────────┬──────────────────┘
   │         │         │
┌──▼──┐   ┌──▼──┐   ┌──▼──┐
│ VM1 │   │ VM2 │   │ VM3 │
│vnet0│   │vnet1│   │vnet2│
└─────┘   └─────┘   └─────┘
```

### Libvirt in Production

**Used by:**
- **OpenStack Nova** - Cloud compute service
- **Proxmox VE** - Virtualization platform
- **oVirt/RHEV** - Enterprise virtualization

**Performance Optimization:**
```xml
<!-- CPU pinning -->
<vcpu placement='static' cpuset='0-3'>4</vcpu>
<cputune>
  <vcpupin vcpu='0' cpuset='0'/>
  <vcpupin vcpu='1' cpuset='1'/>
</cputune>

<!-- Huge pages -->
<memoryBacking>
  <hugepages/>
</memoryBacking>

<!-- NUMA awareness -->
<numatune>
  <memory mode='strict' nodeset='0'/>
</numatune>
```

---

## Cloud Provider Comparison

| Provider | Primary Hypervisor | KVM Usage | Key Technology |
|----------|-------------------|-----------|----------------|
| **AWS** | KVM (Nitro) | Core platform | Hardware offload |
| **GCP** | KVM | Core platform | Software optimization |
| **Azure** | Hyper-V + KVM | Limited (Kata, ACI) | Hybrid approach |

**Note:** Cloud providers use KVM as foundation but build extensive custom infrastructure for scale, security, and performance.

---

*Previous: [05_Container_Technologies_and_cgroups.md](05_Container_Technologies_and_cgroups.md)*
*Next: [07_Specialized_Virtualization.md](07_Specialized_Virtualization.md)*
