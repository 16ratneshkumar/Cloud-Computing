# Complete Box-Type Architecture Diagrams
## Deployments 1–30 | Full Stack with Use Cases, Pros & Cons

> **How to read the box diagrams:**
> - Each box = one software/hardware layer
> - Top = highest abstraction (workloads)
> - Bottom = bare metal hardware
> - Side annotations = what each layer does
> - Arrows / labels show cross-layer communication

---

# DEPLOYMENT 1 — Bare Metal → Linux → LXC

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    WORKLOADS / APPLICATIONS                      │
│   [App Process A]    [App Process B]    [App Process C]          │
└────────────┬────────────────┬───────────────────┬───────────────┘
             │                │                   │
┌────────────▼────────────────▼───────────────────▼───────────────┐
│                    LXC CONTAINERS                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Container A  │  │ Container B  │  │    Container C       │  │
│  │ (own rootfs) │  │ (own rootfs) │  │    (own rootfs)      │  │
│  │ own PID ns   │  │ own PID ns   │  │    own PID ns        │  │
│  │ own net ns   │  │ own net ns   │  │    own net ns        │  │
│  │ own mnt ns   │  │ own mnt ns   │  │    own mnt ns        │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│         Managed via: lxc-start / lxc-attach / lxc-ls            │
└────────────────────────────┬────────────────────────────────────┘
                             │  namespaces + cgroups
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL                                  │
│  ┌────────────┐  ┌─────────────┐  ┌──────────┐  ┌──────────┐  │
│  │ cgroups v2 │  │ namespaces  │  │ seccomp  │  │ AppArmor │  │
│  │(CPU/mem    │  │(pid/net/mnt │  │(syscall  │  │(MAC      │  │
│  │ limits)    │  │ /uts/ipc)   │  │ filter)  │  │ policy)  │  │
│  └────────────┘  └─────────────┘  └──────────┘  └──────────┘  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │        Virtual Network Bridge (lxcbr0 / veth pairs)    │    │
│  └─────────────────────────────────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────────┘
                             │  direct hardware access
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────────┐    │
│  │  CPU (x86/   │  │  RAM (shared  │  │  Storage (disk /  │    │
│  │  ARM)        │  │  by all conts)│  │  NVMe / SAN)      │    │
│  └──────────────┘  └───────────────┘  └───────────────────┘    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           Network Interface Card (NIC / bond)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Linux kernel provides **namespaces** (process, network, mount, UTS, IPC) giving each container an isolated view of the system
- **cgroups** enforce CPU, memory, and I/O resource limits per container
- LXC tools (lxc-start, lxc-attach) manage container lifecycle directly without any daemon
- All containers share the **same kernel** — no virtualization overhead

## Use Case
Direct lightweight containerization. Embedded systems, DIY VPS hosting, minimal overhead dev/test environments, and any scenario where a management daemon is unwanted overhead.

## Pros
- Near-native performance (no hypervisor overhead)
- No daemon process required — direct kernel feature usage
- Fine-grained resource control via cgroups
- Minimal memory and CPU overhead per container
- Good for high-density workloads

## Cons
- Shared kernel — a kernel exploit affects all containers
- No built-in clustering or remote API
- Manual lifecycle management (no GUI, no REST API)
- Network setup requires manual veth/bridge configuration
- No live migration capability without additional tooling

---

# DEPLOYMENT 2 — Bare Metal → Linux → LXD → LXC

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│               MANAGEMENT CLIENTS                                │
│  ┌──────────┐  ┌───────────────┐  ┌───────────────────────┐    │
│  │ lxc CLI  │  │  REST API     │  │  Terraform / Ansible  │    │
│  │(lxc launch│  │(HTTPS :8443) │  │  (LXD provider)       │    │
│  │ lxc list)│  │               │  │                       │    │
│  └────┬─────┘  └───────┬───────┘  └──────────┬────────────┘    │
└───────┼────────────────┼───────────────────────┼────────────────┘
        └────────────────┼───────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                    LXD DAEMON (lxd)                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  REST API Server                         │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐    │
│  │  Clustering  │  │  Profiles &  │  │  Image Manager     │    │
│  │  (Raft/dqlite│  │  Config Mgmt │  │  (simplestreams)   │    │
│  └──────────────┘  └──────────────┘  └────────────────────┘    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               STORAGE BACKENDS                          │    │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌────────┐   │    │
│  │  │ ZFS  │  │ Btrfs│  │ LVM  │  │ Ceph │  │  dir   │   │    │
│  │  └──────┘  └──────┘  └──────┘  └──────┘  └────────┘   │    │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              NETWORK BACKENDS                            │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │    │
│  │  │ bridge   │  │  OVN     │  │  macvlan / sriov     │   │    │
│  │  └──────────┘  └──────────┘  └──────────────────────┘   │    │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  creates/manages
┌────────────────────────────▼────────────────────────────────────┐
│                    LXC CONTAINERS                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Container A  │  │ Container B  │  │    Container C       │  │
│  │ Ubuntu 22.04 │  │ Alpine 3.18  │  │    Debian 12         │  │
│  │ veth0→lxdbr0 │  │ veth0→lxdbr0 │  │    veth0→OVN net    │  │
│  │ Storage: ZFS │  │ Storage: ZFS │  │    Storage: Ceph RBD │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL                                  │
│         cgroups v2 │ namespaces │ seccomp │ AppArmor            │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│      CPU │ RAM │ NVMe/SSD │ NIC (10GbE+ for clustering)        │
└─────────────────────────────────────────────────────────────────┘

CLUSTERING MODE (multiple bare metal nodes):
┌────────────────────────────────────────────────────────────────────────────┐
│  NODE 1                    │  NODE 2                    │  NODE 3          │
│  ┌──────────────────────┐  │  ┌──────────────────────┐  │  ┌───────────┐  │
│  │ LXD (cluster leader) │  │  │ LXD (cluster member) │  │  │ LXD       │  │
│  │ dqlite (Raft leader) │←─┼─→│ dqlite (follower)    │←─┼─→│ (member)  │  │
│  └──────────────────────┘  │  └──────────────────────┘  │  └───────────┘  │
│  Containers/VMs            │  Containers/VMs             │  Containers/VMs │
│  BARE METAL                │  BARE METAL                 │  BARE METAL     │
└────────────────────────────────────────────────────────────────────────────┘
```

## How It Works
- LXD daemon sits above LXC and provides a full REST API, clustering via dqlite (distributed SQLite with Raft consensus), image management, and storage/network abstraction
- Clients (CLI, Terraform, REST) talk to LXD daemon
- LXD translates API calls into LXC container operations
- In cluster mode, dqlite replicates state across nodes; containers can be migrated between nodes

## Use Case
Production VPS hosting, private container clouds, developer environments needing API-driven management. Popular with Canonical-ecosystem users. Good Proxmox alternative for container-first workloads.

## Pros
- Full REST API and CLI management
- Clustering with live migration
- Multiple storage drivers (ZFS recommended)
- Both containers and VMs (QEMU) from one tool
- Image registry with simplestreams

## Cons
- Canonical controls LXD; community forked as "Incus"
- Not Kubernetes-native — separate ecosystem
- Less tooling than Docker/K8s ecosystem
- Security namespace sharing still applies

---

# DEPLOYMENT 3 — Bare Metal → Linux → systemd-nspawn

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTAINERIZED SERVICES                       │
│  ┌──────────────────┐    ┌───────────────────────────────────┐  │
│  │ Service A        │    │ Service B                         │  │
│  │ (own rootfs dir) │    │ (own rootfs dir)                  │  │
│  │ /var/lib/machines│    │ /var/lib/machines/serviceB        │  │
│  │ /serviceA        │    │                                   │  │
│  └──────────────────┘    └───────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                 systemd-nspawn + systemd-machined                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  systemd-nspawn  (chroot-like container runtime)         │   │
│  │  machinectl      (list/start/stop/login to containers)   │   │
│  │  systemd-machined(machine registry daemon)               │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │             Networking                                   │   │
│  │  systemd-networkd │ veth pairs │ host0 interface         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      systemd (PID 1)                            │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │  journald  │  │  networkd    │  │  resolved              │  │
│  └────────────┘  └──────────────┘  └────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      LINUX KERNEL                               │
│              cgroups │ namespaces │ seccomp                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│            CPU │ RAM │ Storage │ NIC                           │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- systemd-nspawn creates lightweight containers using Linux namespaces — essentially a chroot with namespace isolation
- systemd-machined provides a D-Bus API for container lifecycle
- machinectl is the CLI interface
- No external daemon needed — built into every modern Linux

## Use Case
Minimal service isolation, OS image testing, distribution package building, and lightweight dev environments where Docker/LXD is too heavy.

## Pros
- Zero additional software — built into systemd
- Very lightweight
- Good for testing OS images and chroots
- Integrates with journald for container logging

## Cons
- Very limited networking features
- No clustering or live migration
- Not suitable for production multi-tenant workloads
- No image registry or management plane
- Network configuration manual

---

# DEPLOYMENT 4 — Bare Metal → FreeBSD → Jails

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    JAIL WORKLOADS                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Jail A     │  │   Jail B     │  │   Jail C (VNET)      │  │
│  │ (thin jail)  │  │ (thick jail) │  │ (own network stack)  │  │
│  │ shared base  │  │ full copy    │  │ own routing table    │  │
│  │ /usr/jails/A │  │ /usr/jails/B │  │ own firewall (pf)    │  │
│  │ IP: 10.0.0.1 │  │ IP: 10.0.0.2 │  │ IP: 192.168.10.1    │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│               JAIL MANAGEMENT TOOLS                             │
│  ┌──────────────┐  ┌───────────┐  ┌────────────────────────┐   │
│  │  /etc/jail   │  │  iocage   │  │  ezjail                │   │
│  │  .conf       │  │  (Python  │  │  (shell scripts)       │   │
│  │  (native)    │  │   mgmt)   │  │                        │   │
│  └──────────────┘  └───────────┘  └────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    FREEBSD KERNEL                               │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐   │
│  │  Jail Subsystem  │  │  pf (firewall)   │  │  ZFS        │   │
│  │  (kernel jails)  │  │  (per jail ACL)  │  │  (datasets  │   │
│  │                  │  │                  │  │   per jail) │   │
│  └──────────────────┘  └──────────────────┘  └─────────────┘   │
│  ┌──────────────────┐  ┌──────────────────────────────────┐    │
│  │  VNET (Virtual   │  │  MAC framework (Mandatory Access  │    │
│  │  Network Stack)  │  │  Control)                        │    │
│  └──────────────────┘  └──────────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  CPU │ RAM │ ZFS Pool (mirror/RAIDZ) │ NIC (em0/igb0) │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- FreeBSD kernel's jail subsystem creates isolated environments at the kernel level — process, filesystem, and network isolated
- VNET jails get their own complete network stack (routing table, firewall, interfaces)
- ZFS datasets are assigned per jail for storage isolation
- pf firewall enforces network rules between jails and to the outside
- iocage or ezjail provide management tooling on top of native jail commands

## Use Case
Secure multi-tenant hosting on FreeBSD, web hosting panels (like HestiaCP on BSD), firewall appliances, and security-focused deployments. Pre-dates Linux containers by years.

## Pros
- Very mature and battle-tested (since FreeBSD 4.0, year 2000)
- VNET provides full network stack isolation per jail
- ZFS per-jail dataset isolation is excellent
- Strong security model with MAC framework
- Minimal overhead

## Cons
- FreeBSD only — Linux binaries need Linux compat layer
- Limited Linux container tooling ecosystem
- No native Kubernetes integration
- iocage has had maintenance issues
- Smaller community than Linux containers

---

# DEPLOYMENT 5 — Bare Metal → OpenBSD → VMM

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    GUEST VIRTUAL MACHINES                       │
│  ┌──────────────────────────┐  ┌───────────────────────────┐   │
│  │   Linux Guest VM         │  │   OpenBSD Guest VM        │   │
│  │   (Ubuntu / Debian)      │  │   (OpenBSD 7.x)           │   │
│  │   virtio-net / virtio-blk│  │   virtio drivers          │   │
│  │   vio0 (network if)      │  │   vio0 (network if)       │   │
│  └──────────────────────────┘  └───────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  virtio I/O
┌────────────────────────────▼────────────────────────────────────┐
│              OpenBSD VMM (vmm(4) + vmd(8))                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  vmm(4) — kernel VMM driver (Intel VT-x / AMD-V)          │ │
│  │  vmd(8) — userspace VMM management daemon                 │ │
│  │  vmctl  — CLI to create/start/stop VMs                    │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OpenBSD HOST OS                                     │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │  pf (firewall)  │  │  pledge(2)   │  │  unveil(2)         │ │
│  │  (NAT/filter for│  │  (syscall    │  │  (filesystem       │ │
│  │   VM traffic)   │  │   whitelist) │  │   visibility)      │ │
│  └─────────────────┘  └──────────────┘  └────────────────────┘ │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vether(4) / bridge(4) — VM virtual networking           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                 OpenBSD KERNEL                                   │
│  vmm(4) driver │ W^X enforcement │ ASLR │ Stack Canaries        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   Intel VT-x / AMD-V CPU │ RAM │ SSD/HDD │ NIC (em0/bge0)      │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- vmm(4) is the kernel VMM driver using Intel VT-x or AMD-V
- vmd(8) is the userspace management daemon that vmd launches VMs and proxies I/O
- VMs use virtio drivers for network and disk
- pf handles NAT and filtering for VM traffic
- OpenBSD's pledge/unveil further restrict vmd's capabilities

## Use Case
Running Linux services (web servers, databases) behind OpenBSD's security hardening and pf firewall. Security researchers and high-security network appliances.

## Pros
- OpenBSD's world-class security applies to hypervisor
- Minimal, auditable codebase
- pf integration for strict VM network control
- Good for security-conscious labs

## Cons
- No Windows guest support
- No live migration
- Limited guest device support
- Small community — limited documentation
- Performance not competitive with KVM

---

# DEPLOYMENT 6 — Bare Metal → NetBSD → Xen/bhyve

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    GUEST VMs                                    │
│  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │  Xen DomU (PV Guest)   │  │  bhyve VM (HVM Guest)        │  │
│  │  Linux / NetBSD        │  │  Linux / NetBSD              │  │
│  └────────────────────────┘  └──────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│           Xen Dom0 (NetBSD) OR bhyve (NetBSD)                   │
│  ┌──────────────────────────┐  ┌───────────────────────────┐   │
│  │  Xen Dom0 (control)      │  │  bhyve (NetBSD port)      │   │
│  │  NetBSD kernel w/ pv drv │  │  vmm(4) on NetBSD         │   │
│  └──────────────────────────┘  └───────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                     NetBSD HOST KERNEL                          │
│   Xen PV drivers │ virtio drivers │ NPF firewall │ ZFS/FFS     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   CPU (x86 / ARM / SPARC / MIPS) │ RAM │ Storage │ NIC         │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Research and exotic hardware virtualization. NetBSD runs on 100+ platforms; used for porting experiments and academic research.

## Pros
- Extreme hardware portability
- Good for research on non-standard platforms

## Cons
- Very niche — limited production use
- Small community and documentation

---

# DEPLOYMENT 7 — Bare Metal → KVM → QEMU VMs

## Box Architecture

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

---

# DEPLOYMENT 8 — Bare Metal → Linux → libvirt → KVM VMs

## Box Architecture

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

---

# DEPLOYMENT 9 — Bare Metal → Xen → Guest VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              UNPRIVILEGED GUEST VMs (DomU)                      │
│  ┌─────────────────────────────┐  ┌────────────────────────┐   │
│  │   DomU (PV Guest)           │  │   DomU (HVM Guest)     │   │
│  │   Paravirtualized Linux      │  │   Windows / unmodified │   │
│  │   xen-netfront (NIC driver) │  │   QEMU device emulation│   │
│  │   xen-blkfront (disk driver)│  │   Hardware virtualized │   │
│  │   Direct hypercalls to Xen  │  │   with VT-x / AMD-V    │   │
│  └─────────────────────────────┘  └────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  event channels / grant tables
┌────────────────────────────▼────────────────────────────────────┐
│              PRIVILEGED DOMAIN (Dom0)                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Linux (or NetBSD) Dom0 — control domain                 │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐ │   │
│  │  │  xl toolstack│  │  xenstored   │  │  xenconsoled   │ │   │
│  │  │  (xl list,   │  │  (config     │  │  (VM console   │ │   │
│  │  │   xl create) │  │   database)  │  │   management)  │ │   │
│  │  └──────────────┘  └──────────────┘  └────────────────┘ │   │
│  │  ┌──────────────┐  ┌──────────────────────────────────┐  │   │
│  │  │  netback     │  │  blkback                         │  │   │
│  │  │  (network    │  │  (disk backend driver —           │  │   │
│  │  │   backend)   │  │   serves disk to DomU)           │  │   │
│  │  └──────────────┘  └──────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  dom0 is privileged via Xen grants
┌────────────────────────────▼────────────────────────────────────┐
│                  XEN HYPERVISOR                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Xen runs BELOW the OS — loaded by bootloader (GRUB)     │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌───────────┐   │   │
│  │  │  vCPU Scheduler│  │  Memory Mgmt   │  │  Event    │   │   │
│  │  │  (credit2)     │  │  (ballooning,  │  │  Channels │   │   │
│  │  │                │  │   grant tables)│  │           │   │   │
│  │  └────────────────┘  └────────────────┘  └───────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Xen loaded before any OS
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  ┌──────────────────┐  ┌────────────────┐  ┌────────────────┐  │
│  │  CPU (VT-x/AMD-V │  │  RAM           │  │  Disk / SAN    │  │
│  │  for HVM, plain  │  │                │  │                │  │
│  │  CPU for PV)     │  │                │  │                │  │
│  └──────────────────┘  └────────────────┘  └────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  NIC (Dom0 owns physical NIC → bridges to DomU vifs)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Xen loads before any OS via GRUB — it sits directly on hardware
- Dom0 (privileged Linux) boots first and manages all hardware drivers
- DomU guests communicate with Dom0 via paravirtual drivers (event channels, grant tables, shared memory)
- PV guests make hypercalls to Xen directly; HVM guests use hardware virtualization + QEMU emulation
- Dom0's netback/blkback drivers handle I/O on behalf of guest VMs

## Use Case
Type-1 hypervisor for hosting providers, security-sensitive deployments, and historically AWS EC2. Strong isolation between VMs.

## Pros
- True Type-1 — runs before any OS
- Strong isolation — Dom0 is separate from guests
- Mature — 20+ years of production use
- PV paravirtualization offers very good I/O performance

## Cons
- Dom0 is single point of control (if Dom0 fails, all VMs impacted)
- More complex architecture than KVM
- Declining adoption vs KVM
- PV guests need Xen-aware kernels

---

# DEPLOYMENT 10 — Bare Metal → VMware ESXi → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT LAYER                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vCenter Server (separate appliance/VM)                  │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │   │
│  │  │  vSphere    │  │  vMotion     │  │  DRS            │ │   │
│  │  │  Web Client │  │  (live VM    │  │  (auto workload │ │   │
│  │  │  (HTML5)    │  │   migration) │  │   balancing)    │ │   │
│  │  └─────────────┘  └──────────────┘  └─────────────────┘ │   │
│  │  ┌──────────────┐  ┌──────────────────────────────────┐  │   │
│  │  │  HA (High    │  │  vSAN (virtual SAN storage)      │  │   │
│  │  │  Availability│  │  NSX (software-defined network)  │  │   │
│  │  └──────────────┘  └──────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  manages multiple ESXi hosts
┌────────────────────────────▼────────────────────────────────────┐
│                  VMware ESXi HOST                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                 GUEST VMs                                │   │
│  │  ┌─────────────────┐  ┌─────────────────┐              │   │
│  │  │ Windows Server  │  │ Linux VM         │              │   │
│  │  │ VMware Tools    │  │ open-vm-tools    │              │   │
│  │  │ VMXNET3 (NIC)   │  │ VMXNET3 (NIC)   │              │   │
│  │  │ PVSCSI (disk)   │  │ PVSCSI (disk)   │              │   │
│  │  └─────────────────┘  └─────────────────┘              │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VMkernel (ESXi Kernel)                                  │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌───────────┐  │   │
│  │  │  CPU Scheduler │  │  Memory Mgmt   │  │  Storage  │  │   │
│  │  │  (resource     │  │  (TPS, balloon,│  │  Stack    │  │   │
│  │  │   pools)       │  │  swap)         │  │  (vmfs)   │  │   │
│  │  └────────────────┘  └────────────────┘  └───────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Virtual Switch (vSwitch / Distributed vSwitch)   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  VMware-certified hardware (HCL)                                │
│  CPU (Intel Xeon / AMD EPYC) │ RAM │ VMFS Datastore │ 10GbE NIC│
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- ESXi (VMkernel) installs directly on bare metal and provides VMs to the management layer
- VMware Tools / open-vm-tools inside VMs provide paravirtual drivers and guest integration
- vCenter manages multiple ESXi hosts, enabling vMotion (live migration), DRS (load balancing), and HA (failover)
- vSAN pools local disks across ESXi hosts into a shared distributed datastore
- NSX provides software-defined networking across the cluster

## Use Case
Dominant enterprise VM platform. Windows workloads, Oracle DB, SAP. Enterprises needing mature HA, live migration, and full commercial support.

## Pros
- Most feature-rich enterprise hypervisor
- Excellent Windows workload support
- vMotion for zero-downtime maintenance
- DRS auto-balances VM placement
- Enterprise support and compliance certifications

## Cons
- Very expensive licensing (especially post-Broadcom 2024)
- Closed source — no community contribution
- Vendor lock-in
- Broadcom's licensing changes have driven many migrations away

---

# DEPLOYMENT 11 — Bare Metal → Hyper-V → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT LAYER                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  Windows Admin   │  │  Azure Arc       │  │  System      │  │
│  │  Center (WAC)    │  │  (Azure portal   │  │  Center VMM  │  │
│  │  (web UI)        │  │   management)    │  │  (SCVMM)     │  │
│  └──────────┬───────┘  └────────┬─────────┘  └───────┬──────┘  │
└─────────────┼───────────────────┼────────────────────┼─────────┘
              └───────────────────┼────────────────────┘
                                  │  WMI / PowerShell / WinRM
┌─────────────────────────────────▼───────────────────────────────┐
│                   PARENT PARTITION (Management OS)               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Windows Server 2022 (Parent Partition)                  │   │
│  │  Has privileged access to all hardware via Hyper-V VSP   │   │
│  │  ┌──────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │  Hyper-V Manager │  │  PowerShell Hyper-V module   │  │   │
│  │  │  (GUI)           │  │  (Get-VM, New-VM, etc.)      │  │   │
│  │  └──────────────────┘  └──────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │         CHILD VIRTUAL MACHINES (Guest VMs)               │   │
│  │  ┌───────────────────┐  ┌───────────────────────────┐   │   │
│  │  │  Windows Server VM│  │  Linux VM (Gen 2)         │   │   │
│  │  │  Gen 1 / Gen 2    │  │  Hyper-V integration svcs │   │   │
│  │  │  Synthetic adapters│  │ virtio / Hyper-V drivers  │   │   │
│  │  └───────────────────┘  └───────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  enlightened I/O via VMBus
┌────────────────────────────▼────────────────────────────────────┐
│                  HYPER-V HYPERVISOR                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Type-1 — loads before Windows via bootloader            │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────┐  │   │
│  │  │  VMBus          │  │  vCPU Scheduler │  │  Memory │  │   │
│  │  │  (fast I/O bus  │  │                 │  │  Mgmt   │  │   │
│  │  │   Parent↔Child) │  │                 │  │         │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Intel Xeon / AMD EPYC │ RAM │ Storage │ 10GbE/25GbE NIC       │
│  (Azure Stack HCI certified hardware for HCI deployments)       │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Hyper-V is a Type-1 hypervisor loaded by the Windows bootloader before Windows starts
- Windows Server runs in the "Parent Partition" — a privileged VM with hardware access
- Child VMs (guests) communicate via VMBus — a high-speed paravirtual bus
- Synthetic (enlightened) devices replace emulated hardware for better performance
- Azure Arc extends Azure management to on-premises Hyper-V VMs

## Use Case
Windows-centric enterprises, Active Directory environments, SQL Server clusters, and organizations leveraging Azure hybrid cloud (Azure Arc, Azure Stack HCI).

## Pros
- Free with Windows Server licensing
- Excellent Windows workload support
- Azure Arc hybrid management
- PowerShell automation-friendly

## Cons
- Windows-centric — Linux is secondary
- GUI cluster management requires SCVMM (additional cost)
- Less flexible than KVM for pure Linux workloads

---

# DEPLOYMENT 12 — Bare Metal → Proxmox VE → KVM VMs + LXC Containers

## Box Architecture

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

---

# DEPLOYMENT 13 — Bare Metal → Oracle VM (Xen) → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              ORACLE VM MANAGER                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Oracle VM Manager (web console / CLI)                   │   │
│  │  Oracle Enterprise Manager integration                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              ORACLE VM SERVER (Xen-based)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Dom0 (Oracle Linux) — control domain                    │   │
│  │  DomU guest VMs:                                         │   │
│  │  ┌─────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │ Oracle Linux VM │  │ Windows Server VM             │  │   │
│  │  │ (Oracle DB)     │  │                               │  │   │
│  │  └─────────────────┘  └──────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  XEN HYPERVISOR                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Oracle-certified hardware │ CPU │ RAM │ SAN/NAS storage        │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Running Oracle Database and Oracle Middleware in Oracle-licensed and Oracle-supported virtualization. Gives Oracle license compliance advantages.

## Pros
- Oracle-supported for Oracle workloads
- Oracle license optimization (CPU counting)
- Xen performance

## Cons
- Oracle-centric; declining market share
- More expensive Oracle support
- Xen base aging vs KVM

---

# DEPLOYMENT 14 — Bare Metal → oVirt → KVM VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  MANAGEMENT CLIENTS                             │
│  ┌──────────────────┐  ┌─────────────────┐  ┌───────────────┐  │
│  │  oVirt Web UI    │  │  REST API        │  │  Ansible      │  │
│  │  (Admin/User     │  │  (v4 API)        │  │  oVirt module │  │
│  │   Portal)        │  │                  │  │               │  │
│  └──────────────────┘  └─────────────────┘  └───────────────┘  │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│              oVirt ENGINE (Management Server)                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ovirt-engine (Java EE application — JBoss/WildFly)      │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │   │
│  │  │  Scheduler  │  │  Network Mgmt│  │  Storage Mgmt   │ │   │
│  │  │  (VM        │  │  (logical    │  │  (storage        │ │   │
│  │  │   placement)│  │   networks,  │  │   domains: NFS,  │ │   │
│  │  │             │  │   VLANs)     │  │   iSCSI, FC,    │ │   │
│  │  └─────────────┘  └──────────────┘  │   Ceph RBD)     │ │   │
│  │  ┌──────────────────────────────┐   └─────────────────┘ │   │
│  │  │  HA / Migration / Cluster    │                        │   │
│  │  └──────────────────────────────┘                        │   │
│  │  PostgreSQL (config/state DB)                            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────┘
                         │  VDSM agent on each host
┌────────────────────────▼────────────────────────────────────────┐
│              oVirt HOSTS (KVM hypervisor nodes)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VDSM (oVirt host agent) — manages KVM on this host      │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │               KVM Virtual Machines                  │ │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │ │   │
│  │  │  │ VM A         │  │ VM B         │  │ VM C     │  │ │   │
│  │  │  │ Windows      │  │ Linux        │  │ Linux    │  │ │   │
│  │  │  └──────────────┘  └──────────────┘  └──────────┘  │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│              KVM + QEMU + LINUX KERNEL                          │
│   kvm.ko │ libvirt │ TUN/TAP │ Open vSwitch                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                       BARE METAL                                │
│         CPU (VT-x/AMD-V) │ RAM │ Shared Storage │ NIC          │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- oVirt Engine (Java EE app) runs as the central management server with a PostgreSQL database
- VDSM (Virtual Desktop and Server Manager) agent runs on each KVM host and implements oVirt's host API
- Engine communicates with VDSM via XML-RPC to manage VMs on each host
- Storage domains (NFS/iSCSI/FC) are shared across hosts for VM live migration

## Use Case
Open-source alternative to VMware vSphere. Used by organizations wanting enterprise VM management without VMware licensing costs. Upstream of Red Hat Virtualization.

## Pros
- Free, enterprise-grade VM management
- Live migration, HA, storage management
- REST API for automation

## Cons
- Red Hat Virtualization (commercial oVirt) went EOL June 2024
- oVirt community maintenance uncertain
- Complex initial setup
- Proxmox is often chosen over oVirt now

---

# DEPLOYMENT 15 — Bare Metal → XCP-ng → Xen → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              XEN ORCHESTRA (Web Management)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Xen Orchestra (XO) — Node.js web application            │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │   │
│  │  │  VM Mgmt    │  │  Backup &    │  │  Cloud-init     │ │   │
│  │  │  (create,   │  │  Replication │  │  integration    │ │   │
│  │  │  migrate,   │  │  (XO Lite or │  │                 │ │   │
│  │  │  snapshots) │  │   XO Pro)    │  │                 │ │   │
│  │  └─────────────┘  └──────────────┘  └─────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  XAPI protocol
┌────────────────────────────▼────────────────────────────────────┐
│              XCP-ng HOST                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Dom0 (CentOS-based control domain)                       │   │
│  │  ┌────────────────┐  ┌──────────────────────────────┐    │   │
│  │  │  XAPI (toolstack│  │ xcp-networkd (network mgmt) │    │   │
│  │  │  — XAPI daemon) │  └──────────────────────────────┘    │   │
│  │  └────────────────┘                                       │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │             GUEST VMs (DomU)                       │   │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌─────────┐  │   │   │
│  │  │  │ Windows VM   │  │ Linux VM     │  │ BSD VM  │  │   │   │
│  │  │  │ HVM + tools  │  │ PV / HVM     │  │ HVM     │  │   │   │
│  │  │  └──────────────┘  └──────────────┘  └─────────┘  │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │  Storage Repositories (SR)                         │   │   │
│  │  │  LVM SR │ NFS SR │ iSCSI SR │ ZFS SR (community)  │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  XEN HYPERVISOR                                  │
│  vCPU Scheduler (credit2) │ Memory Management │ Event Channels  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   CPU (VT-x/AMD-V) │ RAM │ Local SSD / SAN / NAS │ NIC          │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Open-source Citrix XenServer alternative. Community-driven enterprise Xen platform managed by Xen Orchestra. Popular VMware/Citrix alternative in Europe.

## Pros
- Free open-source Xen platform
- Xen Orchestra provides excellent management UI
- Compatible with Citrix XenServer API and tools
- Good Windows VM support

## Cons
- Xen architecture adds Dom0 complexity
- Smaller ecosystem than KVM-based platforms
- Xen declining vs KVM long-term

---

# DEPLOYMENT 16 — Bare Metal → bhyve (FreeBSD) → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    GUEST VIRTUAL MACHINES                       │
│  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │   Linux Guest          │  │   Windows Guest              │  │
│  │   virtio-net           │  │   AHCI / e1000 emulation     │  │
│  │   virtio-blk           │  │   VirtIO drivers (optional)  │  │
│  └────────────────────────┘  └──────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│         BHYVE + VM MANAGEMENT                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  bhyve (userspace VMM process per VM)                    │   │
│  │  bhyvectl — VM management CLI                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────┐  ┌──────────────────────────────────┐    │
│  │  vm-bhyve         │  │  CBSD                           │    │
│  │  (shell mgmt tool)│  │  (management framework)         │    │
│  └──────────────────┘  └──────────────────────────────────┘    │
│  TrueNAS SCALE uses bhyve for VM hosting alongside ZFS          │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/vmm
┌────────────────────────────▼────────────────────────────────────┐
│                FREEBSD KERNEL (with vmm.ko)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vmm.ko (VMX/SVM support) │ ZFS │ pf │ jails            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   Intel VT-x / AMD-V CPU │ RAM │ ZFS Pool (NVMe/HDD) │ NIC     │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
BSD-native hypervisor. Used in TrueNAS SCALE (NAS + VMs on same hardware), pfSense/OPNsense environments, and FreeBSD development shops.

## Pros
- Native FreeBSD — ZFS + pf + bhyve in one OS
- TrueNAS SCALE provides turnkey NAS + VMs
- Good for homelab NAS with VMs

## Cons
- FreeBSD ecosystem only
- Windows support needs extra drivers
- Less mature than KVM

---

# DEPLOYMENT 17 — Bare Metal → OpenBSD VMM → VMs (Standalone)

*(Architecture diagram already covered in Deployment 5)*

---

# DEPLOYMENT 18 — Bare Metal → QEMU (Software Emulation, No KVM)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              CROSS-ARCHITECTURE GUEST VM                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ARM64 Guest OS (on x86 host)                            │   │
│  │  OR: RISC-V Guest │ MIPS Guest │ PowerPC Guest           │   │
│  │  Full software-emulated architecture                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  fully emulated (no hardware virt)
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU (PURE SOFTWARE EMULATION)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  qemu-system-aarch64 / qemu-system-riscv64 / etc.        │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌───────────┐   │   │
│  │  │  TCG Engine    │  │  Device Models │  │ BIOS/UEFI │   │   │
│  │  │  (Tiny Code    │  │  (virtio /     │  │ Firmware  │   │   │
│  │  │   Generator —  │  │  emulated AHCI,│  │           │   │   │
│  │  │  JIT translate │  │  e1000, etc.)  │  │           │   │   │
│  │  │  guest→host    │  │                │  │           │   │   │
│  │  │  instructions) │  │                │  │           │   │   │
│  │  └────────────────┘  └────────────────┘  └───────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  runs as userspace process
┌────────────────────────────▼────────────────────────────────────┐
│                 HOST OS (Linux / macOS / Windows)                │
│   No KVM module needed — pure userspace emulation               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   Any CPU architecture — hardware virtualization NOT required   │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Cross-architecture development and testing. Embedded firmware development (testing ARM firmware on x86 workstation), kernel developers testing cross-arch changes, CI pipelines building ARM packages on x86.

## Pros
- Runs on any hardware — no VT-x/AMD-V needed
- Full architecture emulation (ARM, RISC-V, MIPS, SPARC)
- Excellent for embedded development

## Cons
- Very slow (10-100x slower than native)
- Not suitable for production
- High CPU overhead

---

# DEPLOYMENT 19 — IBM Z Hardware → z/VM → Linux/zOS Guests

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  TENANT WORKLOADS                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│  │  z/OS      │  │  Linux on  │  │  Linux on  │  │  z/VSE   │  │
│  │  LPAR      │  │  Z (RHEL)  │  │  Z (Ubuntu)│  │  LPAR    │  │
│  │  (COBOL,   │  │  (Java,    │  │  (DB2,     │  │  (legacy)│  │
│  │  DB2, CICS)│  │  Python)   │  │  WebSphere)│  │          │  │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │
│         (thousands of Linux guests possible on one system)      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  z/VM HYPERVISOR                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  z/VM  (CP — Control Program)                            │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────┐  │   │
│  │  │  Virtual Machine│  │  Guest Memory   │  │  DASD   │  │   │
│  │  │  Scheduler (SRM)│  │  Management     │  │  Mgmt   │  │   │
│  │  │                 │  │                 │  │         │  │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────┘  │   │
│  │  ┌──────────────────────────────────────────────────────┐│   │
│  │  │  CMS (Conversational Monitor System) — admin env    ││   │
│  │  └──────────────────────────────────────────────────────┘│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              IBM Z HARDWARE (z16 / z15 / LinuxONE)              │
│  ┌───────────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │  IBM z CPU        │  │  RAM          │  │  DASD / Flash   │  │
│  │  (S390X arch)     │  │  (up to 32TB  │  │  Express / DS8k │  │
│  │  PU, IFL, ICF,    │  │   on z16)     │  │  storage        │  │
│  │  SAP, zIIP cpus   │  │               │  │                 │  │
│  └───────────────────┘  └───────────────┘  └─────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RAS: Redundant CPUs/memory, chip-spare, fault isolation  │   │
│  │  Pervasive Encryption, Secure Boot                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Banking (SWIFT, ACH transactions), insurance (batch processing), government, and healthcare — where 99.9999% uptime and extremely high VM density are mandatory.

## Pros
- Six nines availability
- Runs thousands of VMs per system
- Hardware memory encryption by default
- Unmatched I/O for transaction processing
- 50+ years of refinement

## Cons
- Extremely expensive (millions USD)
- IBM ecosystem lock-in
- Requires specialized mainframe expertise
- z/OS skills are rare

---

# DEPLOYMENT 20 — IBM Power Systems → PowerVM → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOGICAL PARTITIONS (LPARs)                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐   │
│  │  AIX LPAR   │  │  Linux LPAR │  │  IBM i LPAR          │   │
│  │  (Oracle DB,│  │  (SAP HANA, │  │  (AS/400 workloads)  │   │
│  │   WebSphere)│  │   Red Hat)  │  │                      │   │
│  └─────────────┘  └─────────────┘  └──────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  PowerVM HYPERVISOR                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  POWER Hypervisor (PHYP) — embedded in firmware          │   │
│  │  ┌──────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │  Micro-partitions│  │  Shared Processor Pools      │  │   │
│  │  │  (0.1 vCPU       │  │  (dynamic CPU allocation)    │  │   │
│  │  │  granularity)    │  │                              │  │   │
│  │  └──────────────────┘  └──────────────────────────────┘  │   │
│  │  ┌──────────────────────────────────────────────────────┐ │   │
│  │  │  Virtual I/O Server (VIOS) — storage/net for LPARs  │ │   │
│  │  └──────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│  HMC (Hardware Management Console) — external management node   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              IBM POWER HARDWARE (POWER9/POWER10)                │
│  ┌──────────────────┐  ┌────────────────┐  ┌────────────────┐  │
│  │  POWER CPU       │  │  RAM (up to    │  │  NVMe / SAN    │  │
│  │  (POWER10 arch)  │  │  8TB per node) │  │  storage       │  │
│  └──────────────────┘  └────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
SAP HANA, Oracle Database, and AIX mission-critical workloads requiring POWER architecture's memory bandwidth and RAS.

## Pros
- Superior memory bandwidth for in-memory databases
- AIX and IBM i native support
- Micro-partitioning (0.1 vCPU granularity)

## Cons
- Very expensive IBM POWER hardware
- Specialized expertise scarce

---

# DEPLOYMENT 21 — Embedded Hardware → QNX Hypervisor → RTOS + Linux

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│           MIXED-CRITICALITY GUEST PARTITIONS                    │
│  ┌────────────────────────────┐  ┌───────────────────────────┐  │
│  │  SAFETY PARTITION (RT)     │  │  GENERAL PURPOSE PARTITION│  │
│  │  QNX RTOS                  │  │  Linux / Android          │  │
│  │  ┌──────────┐  ┌────────┐  │  │  ┌───────────┐  ┌──────┐ │  │
│  │  │ Brake    │  │ Power  │  │  │  │Infotainment│  │  OTA │ │  │
│  │  │ Control  │  │ Mgmt   │  │  │  │ (Android)  │  │Update│ │  │
│  │  └──────────┘  └────────┘  │  │  └───────────┘  └──────┘ │  │
│  │  Hard RT guarantees        │  │  Soft RT / best-effort    │  │
│  │  ISO 26262 ASIL-D          │  │  No safety cert required  │  │
│  └────────────────────────────┘  └───────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │  inter-partition communication
┌────────────────────────────▼────────────────────────────────────┐
│              QNX HYPERVISOR                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Temporal Isolation (CPU time guaranteed per partition)   │   │
│  │  Spatial Isolation (memory pages protected per partition) │   │
│  │  Virtual device model (virtio for IPC between partitions) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              EMBEDDED SoC (Intel / ARM / Qualcomm)              │
│  ┌────────────────┐  ┌────────────────┐  ┌───────────────────┐  │
│  │  CPU cores     │  │  RAM           │  │  Peripherals      │  │
│  │  (multi-core   │  │  (partitioned  │  │  (CAN bus, LIN,  │  │
│  │   heterogeneous│  │   between      │  │   PCIe, MIPI,    │  │
│  │   optional)    │  │   partitions)  │  │   Ethernet)       │  │
│  └────────────────┘  └────────────────┘  └───────────────────┘  │
│                  BARE METAL (SoC)                                │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Automotive (digital instrument clusters, ADAS), industrial automation, medical devices requiring mixed-criticality — real-time safety alongside general-purpose Linux.

## Pros
- Hard RT guarantees for safety partition
- ISO 26262 ASIL-D certified
- Temporal and spatial isolation proven in automotive OEMs

## Cons
- Expensive QNX licensing
- Specialized expertise required
- Closed ecosystem

---

# DEPLOYMENT 22 — Embedded Hardware → INTEGRITY Multivisor

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│           CERTIFIED SAFETY PARTITIONS                           │
│  ┌───────────────────────────┐  ┌──────────────────────────┐   │
│  │  INTEGRITY RTOS Partition │  │  Linux/Android Partition │   │
│  │  (DO-178C Level A)        │  │  (general purpose)       │   │
│  │  Flight control / avionics│  │  Mission systems UI      │   │
│  └───────────────────────────┘  └──────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              INTEGRITY Multivisor (Green Hills Software)         │
│  Formally certified to DO-178C Level A / IEC 62443              │
│  Proven zero kernel vulnerabilities in 30+ years                │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│    Safety-Critical Embedded Hardware (Aerospace / Defense)      │
│  CPU (PowerPC / ARM / x86) │ RAM │ Custom I/O                   │
│                  BARE METAL                                     │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Commercial aircraft (Boeing 787 fly-by-wire), military systems, and medical devices requiring the highest safety certifications.

## Pros / Cons — See previous section (Deployment 22 detailed earlier)

---

# DEPLOYMENT 23 — Embedded Hardware → PikeOS → Rail/Avionics VMs

## Box Architecture

```
┌──────────────────────────────────────────────────────────┐
│         SAFETY + GENERAL PURPOSE PARTITIONS              │
│  ┌────────────────────┐  ┌───────────────────────────┐   │
│  │  POSIX RTOS Task   │  │  Linux Guest              │   │
│  │  (EN 50128 SIL 4   │  │  (maintenance/diagnostics)│   │
│  │   rail signaling)  │  │                           │   │
│  └────────────────────┘  └───────────────────────────┘   │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│          PikeOS (SYSGO) — certified partitioning RTOS    │
│  EN 50128 SIL 4 │ DO-178C │ IEC 61508                    │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│     Embedded Hardware (Rail / Avionics)                  │
│  PowerPC / ARM │ Radiation-hardened CPUs possible        │
│                  BARE METAL                              │
└──────────────────────────────────────────────────────────┘
```

## Use Case
Rail signaling systems (ETCS), avionics (ARINC 653), and defense.

---

# DEPLOYMENT 24 — Bare Metal → ACRN Hypervisor → Service VM + RT VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  GUEST PARTITIONS                               │
│  ┌────────────────────────────┐  ┌───────────────────────────┐  │
│  │  SERVICE VM (SOS)          │  │  USER VMs (UOS)           │  │
│  │  Ubuntu / Clear Linux      │  │  ┌───────────┐  ┌──────┐  │  │
│  │  ┌─────────────────────┐   │  │  │ RT Linux  │  │ IoT  │  │  │
│  │  │ Device Model (DM)   │   │  │  │ (hard RT  │  │ Work │  │  │
│  │  │ virtio backend devs │   │  │  │ workload) │  │ load │  │  │
│  │  └─────────────────────┘   │  │  └───────────┘  └──────┘  │  │
│  │  Manages UOS lifecycle     │  │                           │  │
│  └────────────────────────────┘  └───────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │  ACRN hypervisor calls
┌────────────────────────────▼────────────────────────────────────┐
│                  ACRN HYPERVISOR                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VMX-based Type-1 hypervisor (Intel VT-x only)           │   │
│  │  ┌──────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │  vCPU Scheduler  │  │  Memory Partitioning         │  │   │
│  │  │  (LAPIC passthru │  │  (EPT page tables)           │  │   │
│  │  │  for RT VMs)     │  │                              │  │   │
│  │  └──────────────────┘  └──────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  ACRN loaded by UEFI / bootloader
┌────────────────────────────▼────────────────────────────────────┐
│              INTEL EDGE HARDWARE (x86 IoT / Automotive SoC)    │
│  Intel VT-x required │ RAM │ eMMC / NVMe │ CAN / LTE / Ethernet│
│                  BARE METAL                                     │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Intel IoT edge devices, connected vehicles (non-safety-critical), industrial PCs requiring mixed real-time and general-purpose workloads with open-source tooling.

## Pros
- Open source (Linux Foundation)
- Designed for Intel edge silicon
- Mixed-criticality support
- Smaller footprint than server hypervisors

## Cons
- Intel x86 only
- Smaller community than KVM
- Not for general-purpose cloud use

---

# DEPLOYMENT 25 — Bare Metal → Nutanix AHV → VMs + Integrated Storage

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  MANAGEMENT PLANE                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Prism Central (multi-cluster management)                │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  VM Mgmt │ Storage Mgmt │ Networking │ Flow (SDN)   │ │   │
│  │  │  Leap (DR) │ X-Play (automation) │ Calm (IaC)      │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  manages
┌────────────────────────────▼────────────────────────────────────┐
│              NUTANIX NODE (each node has AHV + CVM)             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           AHV HYPERVISOR (KVM-based)                     │   │
│  │   ┌───────────────────────────────────────────────────┐  │   │
│  │   │           GUEST VMs                               │  │   │
│  │   │  ┌──────────────┐  ┌──────────────┐              │  │   │
│  │   │  │  Windows VM  │  │  Linux VM    │              │  │   │
│  │   │  │  virtio-scsi │  │  virtio-net  │              │  │   │
│  │   │  └──────────────┘  └──────────────┘              │  │   │
│  │   └───────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │     CVM (Controller VM) — per node storage controller    │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Stargate (I/O path) │ Curator (distributed ops)│    │   │
│  │  │  Cassandra (metadata ring) │ Zookeeper (coord)  │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  distributed storage fabric
┌────────────────────────────▼────────────────────────────────────┐
│         NUTANIX DISTRIBUTED STORAGE FABRIC (across nodes)       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1 SSD/HDD ──── Node 2 SSD/HDD ──── Node 3 SSD/HDD │   │
│  │           Replication Factor 2 or 3                      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Nutanix-Certified Nodes (NX / Dell XC / HPE DX / Lenovo HX)   │
│  CPU (Intel/AMD) │ RAM (ECC) │ NVMe SSD + HDD │ 10/25GbE NIC   │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Each Nutanix node runs AHV (KVM-based) for VM compute and a CVM (Controller VM) for storage
- The CVM implements Nutanix's distributed storage, presenting a local virtual disk to guest VMs
- Data is replicated across CVM nodes via the Stargate I/O path
- Prism Central manages multiple clusters from a single pane
- Nutanix Flow provides micro-segmentation networking

## Use Case
Enterprise HCI replacing VMware + SAN. VDI, database hosting, and distributed enterprise workloads.

## Pros
- Integrated compute + storage — no external SAN
- Prism Central is excellent management UI
- Self-healing distributed storage
- Hardware-agnostic

## Cons
- Expensive licensing
- AHV less feature-rich than ESXi historically
- Nutanix software lock-in

---

# DEPLOYMENT 26 — Bare Metal → HPE VM Essentials → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              HPE VM ESSENTIALS MANAGEMENT                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  HPE VM Essentials Console (web UI)                      │   │
│  │  HPE OneView integration (hardware lifecycle)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│          KVM-BASED HYPERVISOR (HPE managed)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Guest VMs (Windows / Linux)                             │   │
│  │  KVM + libvirt backend managed by HPE tooling            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  HPE ProLiant / Synergy / Alletra (HPE hardware)                │
│  iLO (integrated Lights Out) management                         │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
HPE hardware customers seeking VMware alternative after Broadcom pricing changes.

---

# DEPLOYMENT 27 — Bare Metal → Huawei FusionCompute → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              FUSIONMANAGER / FUSIONSPHERE                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  FusionManager (cluster management)                      │   │
│  │  FusionSphere OpenStack (cloud layer, optional)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              FUSIONCOMPUTE (CNA + VRM)                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CNA (Compute Node Agent) — KVM hypervisor per node      │   │
│  │  VRM (Virtual Resource Manager) — cluster manager        │   │
│  │  Guest VMs (Windows / Linux / Euler OS)                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Huawei TaiShan (ARM Kunpeng) / FusionServer (x86)              │
│  Huawei OceanStor storage │ CloudEngine network switches        │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Chinese domestic market enterprise cloud. Banks, telecoms, government in China.

---

# DEPLOYMENT 28 — Bare Metal → Scale Computing HC3 → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              SCALE COMPUTING MANAGEMENT                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  HyperCore UI (web management)                           │   │
│  │  REST API │ Scale Computing Fleet Manager (multi-site)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              HyperCore OS (KVM-based HCI)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KVM Virtual Machines (Windows / Linux)                  │   │
│  │  SCRIBE (distributed storage — local NVMe pool)          │   │
│  │  Self-healing storage (auto-rebuild on node failure)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Scale Computing HC3/HC4 Edge Nodes (3-node minimum for prod)  │
│  Intel CPU │ RAM │ NVMe SSD │ 10GbE NIC                        │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Edge computing and ROBO (remote office/branch office) virtualization. Simple HCI for non-IT environments.

---

# DEPLOYMENT 29 — Bare Metal → Kubernetes → Containers

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES CLIENTS                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  kubectl     │  │  Helm        │  │  ArgoCD / FluxCD     │  │
│  │  (CLI)       │  │  (package    │  │  (GitOps operator)   │  │
│  │              │  │   manager)   │  │                      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │  REST / HTTPS (port 6443)
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES CONTROL PLANE                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kube-apiserver — REST API, auth, admission control      │   │
│  │  etcd — distributed key-value store (cluster state)      │   │
│  │  kube-scheduler — assigns pods to nodes                  │   │
│  │  kube-controller-manager — reconciliation loops          │   │
│  │  cloud-controller-manager — optional, cloud integration  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  kubelet on each node
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES WORKER NODES                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1                   │  Node 2         │  Node 3    │   │
│  │  ┌─────────────────────┐  │  ┌───────────┐  │  ┌──────┐  │   │
│  │  │  kubelet (node agent)│  │  │  kubelet  │  │  │kubelt│  │   │
│  │  │  kube-proxy (netw)   │  │  │  kube-    │  │  │kproxy│  │   │
│  │  └─────────────────────┘  │  │  proxy    │  │  └──────┘  │   │
│  │  ┌─────────────────────┐  │  └───────────┘  │            │   │
│  │  │  PODS               │  │                 │            │   │
│  │  │  ┌──────┐  ┌──────┐ │  │  ┌──────────┐  │  ┌──────┐  │   │
│  │  │  │Pod A │  │Pod B │ │  │  │  Pod C   │  │  │ Pod D│  │   │
│  │  │  │(nginx│  │(app) │ │  │  │  (mysql) │  │  │(cache│  │   │
│  │  │  │cont) │  │      │ │  │  │          │  │  │ cont)│  │   │
│  │  │  └──────┘  └──────┘ │  │  └──────────┘  │  └──────┘  │   │
│  │  └─────────────────────┘  │                │            │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ADD-ONS (DaemonSets / System Pods)                      │   │
│  │  ┌────────────┐  ┌──────────┐  ┌─────────┐  ┌────────┐  │   │
│  │  │  CNI plugin│  │  CSI     │  │  CoreDNS│  │ metrics│  │   │
│  │  │  (Calico / │  │  driver  │  │  (DNS)  │  │ server │  │   │
│  │  │   Cilium / │  │  (storage│  │         │  │        │  │   │
│  │  │   Flannel) │  │   plugin)│  │         │  │        │  │   │
│  │  └────────────┘  └──────────┘  └─────────┘  └────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  CRI (containerd / CRI-O)
┌────────────────────────────▼────────────────────────────────────┐
│              CONTAINER RUNTIME (per node)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  containerd (or CRI-O)                                   │   │
│  │  → pulls images from registry                            │   │
│  │  → creates container namespaces                          │   │
│  │  → manages container lifecycle                           │   │
│  │  runc / crun — OCI container runtime (actual container)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (each node)                     │
│  cgroups v2 │ namespaces │ iptables/eBPF │ overlay filesystem   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Node 1          │  Node 2          │  Node 3                  │
│  CPU│RAM│NVMe    │  CPU│RAM│NVMe    │  CPU│RAM│NVMe            │
│  25GbE NIC       │  25GbE NIC       │  25GbE NIC               │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- kube-apiserver is the central REST API; all components talk to it
- etcd stores all cluster state (only control plane writes to it)
- Scheduler watches for unscheduled pods and assigns them to nodes based on resource requests and affinity rules
- kubelet on each node communicates with apiserver and manages pods via containerd
- CNI plugin handles pod networking (assigns IPs, routing between pods across nodes)
- CSI driver handles persistent storage (provisioning PVCs on NFS/Ceph/local)

## Use Case
Cloud-native microservices on bare metal for maximum performance. Financial services, telecoms, and tech companies wanting no VM overhead.

## Pros
- No VM overhead — maximum density
- Declarative GitOps management
- Massive ecosystem (Helm, operators, CNCF tools)
- Auto-scaling, self-healing
- Multi-cloud portable workload definitions

## Cons
- No native VM support (need KubeVirt)
- Bare metal provisioning complex (kubeadm / k3s / Talos)
- etcd requires careful backup and maintenance
- Steep learning curve
- Networking complexity (CNI plugins)

---

# DEPLOYMENT 30 — Bare Metal → k3s → Containers (Lightweight Kubernetes)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBECTL / HELM / FLUX                              │
│  (same Kubernetes API — compatible tooling)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │  port 6443
┌────────────────────────────▼────────────────────────────────────┐
│              k3s SERVER (Control Plane — single binary)         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  /usr/local/bin/k3s  (ALL-IN-ONE ~70MB binary)          │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  kube-apiserver (embedded)                          │ │   │
│  │  │  kube-scheduler (embedded)                          │ │   │
│  │  │  kube-controller-manager (embedded)                 │ │   │
│  │  │  SQLite (default) OR etcd / external DB (HA mode)   │ │   │
│  │  │  containerd (embedded CRI)                          │ │   │
│  │  │  Flannel (embedded CNI — simple VXLAN)              │ │   │
│  │  │  Traefik (embedded ingress controller)              │ │   │
│  │  │  CoreDNS (embedded DNS)                             │ │   │
│  │  │  Local-path-provisioner (embedded storage)          │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  k3s agent on worker nodes
┌────────────────────────────▼────────────────────────────────────┐
│              k3s AGENT NODES (Worker Nodes)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  /usr/local/bin/k3s agent                                │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  kubelet (embedded) │ kube-proxy (embedded)        │  │   │
│  │  │  containerd (embedded CRI)                         │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │             PODS                                   │  │   │
│  │  │  ┌──────────────┐  ┌──────────────┐              │  │   │
│  │  │  │  Pod A       │  │  Pod B       │              │  │   │
│  │  │  │  (app)       │  │  (service)   │              │  │   │
│  │  │  └──────────────┘  └──────────────┘              │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (ARM64 or x86)                  │
│  cgroups │ namespaces │ iptables │ overlayfs                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Raspberry Pi / Intel NUC / ARM SBC / Edge Server              │
│  As low as 512MB RAM per node │ SD card / eMMC / SSD storage    │
└─────────────────────────────────────────────────────────────────┘

k3s HA MODE (3 servers + etcd):
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  k3s Server 1│  │  k3s Server 2│  │  k3s Server 3│
│  (etcd node) │◄─┤  (etcd node) ├─►│  (etcd node) │
└──────────────┘  └──────────────┘  └──────────────┘
       ↕                  ↕                  ↕
┌──────────────────────────────────────────────────┐
│  k3s Agent Nodes (worker nodes)                  │
└──────────────────────────────────────────────────┘
```

## How It Works
- k3s bundles everything (apiserver, scheduler, controller-manager, kubelet, containerd, Flannel CNI, CoreDNS, Traefik) into a single ~70MB binary
- Single node: k3s uses SQLite instead of etcd — dramatically reducing resource usage
- HA mode: k3s switches to embedded etcd across 3+ server nodes
- k3s agent runs on worker nodes — connects back to server via TLS

## Use Case
Edge computing, IoT gateways, Raspberry Pi clusters, resource-constrained environments, dev/test, and remote deployments needing minimal footprint Kubernetes.

## Pros
- Single binary — easy install (curl | bash)
- Works on ARM (Raspberry Pi, Jetson, AWS Graviton)
- Only ~512MB RAM for a basic cluster
- Full Kubernetes API compatible
- Fast startup

## Cons
- SQLite backend not for large HA production
- Some enterprise K8s features absent
- Less tested at hyperscale (1000+ nodes)
- Flannel CNI less feature-rich than Calico/Cilium

---

# QUICK REFERENCE SUMMARY TABLE: Deployments 1–30

| # | Stack | Box Layers (Bottom→Top) | Best For | Complexity |
|---|-------|------------------------|---------|-----------|
| 1 | LXC | Bare Metal → Linux Kernel → LXC containers | Embedded, high-density | Low |
| 2 | LXD | Bare Metal → Linux → LXD daemon → LXC | VPS, SMB containers | Low-Med |
| 3 | systemd-nspawn | Bare Metal → Linux → systemd → nspawn | Dev/test, service isolation | Low |
| 4 | FreeBSD Jails | Bare Metal → FreeBSD kernel → Jails | BSD hosting, security | Low-Med |
| 5 | OpenBSD VMM | Bare Metal → OpenBSD → vmm/vmd → VMs | Security research | Low |
| 6 | NetBSD Xen/bhyve | Bare Metal → NetBSD → Xen/bhyve → VMs | Research, exotic HW | Low |
| 7 | KVM+QEMU | Bare Metal → Linux → KVM → QEMU VMs | Universal VM hosting | Medium |
| 8 | libvirt+KVM | Bare Metal → Linux → KVM → libvirt → VMs | Managed VM hosts | Medium |
| 9 | Xen | Bare Metal → Xen hypervisor → Dom0 → DomU VMs | Hosting, security | Med-High |
| 10 | VMware ESXi | Bare Metal → ESXi → vCenter → VMs | Enterprise Windows | Medium |
| 11 | Hyper-V | Bare Metal → Hyper-V → Parent Partition → VMs | Microsoft shops | Medium |
| 12 | Proxmox | Bare Metal → Proxmox → KVM VMs + LXC containers | SMB, VMware migration | Medium |
| 13 | Oracle VM | Bare Metal → Xen (Oracle) → Oracle VMs | Oracle DB workloads | Medium |
| 14 | oVirt | Bare Metal → KVM → VDSM → oVirt Engine → VMs | Open vSphere alternative | High |
| 15 | XCP-ng | Bare Metal → Xen → XAPI → Xen Orchestra → VMs | Citrix migration | Med-High |
| 16 | bhyve | Bare Metal → FreeBSD → bhyve → VMs | BSD/NAS with VMs | Low-Med |
| 17 | OpenBSD VMM | (see #5) | Security appliances | Low |
| 18 | QEMU only | Bare Metal → Host OS → QEMU (TCG) → any-arch VM | Cross-arch dev/test | Low |
| 19 | IBM z/VM | IBM Z HW → z/VM → z/OS + Linux guests | Banking, government | Very High |
| 20 | PowerVM | IBM Power HW → PowerVM → AIX/Linux LPARs | SAP HANA, Oracle | Very High |
| 21 | QNX Hypervisor | Embedded SoC → QNX → RTOS + Linux partitions | Automotive, medical | High |
| 22 | INTEGRITY | Safety HW → INTEGRITY → certified partitions | Aerospace, defense | Extreme |
| 23 | PikeOS | Rail HW → PikeOS → RTOS + Linux | Rail, avionics | High |
| 24 | ACRN | Intel Edge HW → ACRN → SOS + UOS partitions | IoT, automotive | High |
| 25 | Nutanix AHV | Nutanix Nodes → AOS → AHV → CVM → VMs | Enterprise HCI | Medium |
| 26 | HPE VM Essentials | HPE HW → KVM (HPE) → VMs | HPE ESXi migration | Medium |
| 27 | FusionCompute | Huawei HW → FusionCompute → VMs | China enterprise | Medium |
| 28 | Scale HC3 | Scale Nodes → HyperCore → KVM VMs | Edge/ROBO HCI | Low |
| 29 | Kubernetes | Bare Metal → Linux → containerd → K8s → Pods | Cloud-native apps | High |
| 30 | k3s | Bare Metal (edge) → Linux → k3s → Pods | Edge, IoT, dev | Medium |

---
*Part 1 of 4 — Deployments 1–30 with complete box architectures*
*See Part 2 for Deployments 31–60 | Part 3 for 61–90 | Part 4 for 91–110*
