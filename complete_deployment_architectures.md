# Complete Bare-Metal Virtualization Deployment Reference
### 110 Deployments — Architecture, Use Cases, Pros & Cons

---

> **How to read this document:**
> Each deployment entry contains:
> - **Architecture** — layered stack diagram
> - **Use Case** — who uses it and why
> - **Pros** — strengths
> - **Cons** — weaknesses / limitations

---

# CATEGORY 1: OS-Level Virtualization

---

## Deployment 1 — Bare Metal → Linux → LXC

### Architecture
```
Physical Hardware
 └── Linux Kernel (Host OS)
      └── LXC (Linux Containers)
           ├── Container A (shared kernel)
           ├── Container B (shared kernel)
           └── Container N ...
```

### Use Case
Direct lightweight containerization on Linux without any management daemon. Used in embedded systems, bare-bones hosting environments, and DIY server setups where minimal overhead is the priority.

### Pros
- Extremely low overhead — near-native performance
- No daemon or management layer required
- Fine-grained control via cgroups and namespaces
- Minimal resource footprint

### Cons
- No built-in clustering or remote management
- Manual lifecycle management (no GUI/API)
- Shared kernel — less isolation than VMs
- Security vulnerabilities in one container can affect host

---

## Deployment 2 — Bare Metal → Linux → LXD → LXC

### Architecture
```
Physical Hardware
 └── Linux Kernel (Host OS)
      └── LXD Daemon (REST API + clustering)
           └── LXC Containers
                ├── Container A
                ├── Container B
                └── Container N
```

### Use Case
Production Linux container hosting with management APIs, clustering across multiple nodes, and storage driver abstraction. Popular for VPS providers, private cloud hosting, and dev/test environments needing a simple but powerful container platform.

### Pros
- REST API and CLI management
- Supports clustering across nodes
- Multiple storage backends (ZFS, Btrfs, LVM, Ceph)
- Live migration of containers
- Supports both containers and VMs (via QEMU internally)

### Cons
- Still shares kernel — less isolation than VMs
- Not as feature-rich as Kubernetes for large-scale orchestration
- LXD community forked to Canonical; ecosystem split
- Limited ecosystem compared to Docker/K8s

---

## Deployment 3 — Bare Metal → Linux → systemd-nspawn

### Architecture
```
Physical Hardware
 └── Linux Kernel
      └── systemd
           └── systemd-nspawn Containers
                ├── Container A (chroot-like)
                └── Container B
```

### Use Case
Minimal, built-in Linux containerization using systemd. Used for lightweight service isolation, OS development, testing distribution images, and minimal embedded deployments.

### Pros
- No additional software — built into systemd
- Very lightweight
- Good for testing OS images and distributions
- Integrates with systemd-machined for basic management

### Cons
- Very limited orchestration capabilities
- No native clustering
- Not production-grade for multi-tenant workloads
- Less mature security model than LXC/LXD

---

## Deployment 4 — Bare Metal → FreeBSD → Jails

### Architecture
```
Physical Hardware
 └── FreeBSD Kernel
      └── BSD Jails
           ├── Jail A (isolated userspace)
           ├── Jail B
           └── Jail N
```

### Use Case
Secure OS-level isolation on FreeBSD systems. Used in hosting providers, security-focused environments, and BSD-based network appliances. Predates Linux containers and is considered very mature.

### Pros
- Extremely mature and battle-tested (since 2000)
- Strong security model
- Very low overhead
- Ezjail / iocage tools provide decent management
- Network stack isolation (VNET jails)

### Cons
- BSD-only — Linux workloads need emulation layer
- Smaller ecosystem vs Linux containers
- No native Kubernetes or Docker support
- Limited cloud tooling integrations

---

## Deployment 5 — Bare Metal → OpenBSD → VMM

### Architecture
```
Physical Hardware
 └── OpenBSD Kernel
      └── OpenBSD VMM (vmm/vmd)
           ├── VM A (Linux/OpenBSD guest)
           └── VM B
```

### Use Case
Security-focused virtualization on OpenBSD. Used where extreme host OS security matters — security researchers, firewall/router appliances, hardened infrastructure.

### Pros
- OpenBSD's legendary security hardening applies to hypervisor
- Minimal attack surface
- Simple, auditable codebase
- Good for running Linux guests securely

### Cons
- Limited guest support (no Windows)
- Fewer features than KVM/ESXi
- Small community
- No enterprise tooling
- Performance not competitive with KVM

---

## Deployment 6 — Bare Metal → NetBSD → Xen/bhyve

### Architecture
```
Physical Hardware
 └── NetBSD Kernel
      └── Xen or bhyve
           └── Guest VMs
```

### Use Case
Niche BSD virtualization for portability and research environments. NetBSD runs on almost any hardware; used in embedded research and exotic hardware virtualization.

### Pros
- Extreme portability (NetBSD runs on 100+ platforms)
- Good for research and exotic hardware
- Stable, conservative OS design

### Cons
- Very niche — limited production use
- Small ecosystem
- Not recommended for enterprise workloads
- Limited tooling and documentation

---

# CATEGORY 2: Hardware Hypervisors (Type-1 / Type-2)

---

## Deployment 7 — Bare Metal → KVM → QEMU VMs

### Architecture
```
Physical Hardware (with Intel VT-x / AMD-V)
 └── Linux Kernel
      └── KVM (Kernel Virtual Machine module)
           └── QEMU (machine emulator + virtualizer)
                ├── VM A (Windows / Linux)
                ├── VM B
                └── VM N
```

### Use Case
The most widely deployed open-source virtualization stack. Foundation of most private clouds, hosting providers, and enterprise Linux environments. Used by developers, labs, and production servers worldwide.

### Pros
- Industry standard open-source hypervisor
- Near-native performance with VT-x/AMD-V
- Supports Windows, Linux, BSD, macOS guests
- Huge ecosystem (libvirt, Proxmox, OpenStack, oVirt)
- Active upstream development (Linux kernel)

### Cons
- Requires hardware virtualization support (VT-x/AMD-V)
- Raw KVM/QEMU has no management GUI
- Nested virtualization degrades performance
- Complex networking setup without tooling (libvirt/OVS)

---

## Deployment 8 — Bare Metal → Linux → libvirt → KVM VMs

### Architecture
```
Physical Hardware
 └── Linux Kernel + KVM
      └── libvirt (management layer)
           ├── virsh CLI
           ├── virt-manager GUI
           └── KVM VMs
                ├── VM A
                └── VM B
```

### Use Case
Managed KVM virtualization with a unified API used by OpenStack, oVirt, and virt-manager. Standard for standalone hypervisor hosts managed by administrators.

### Pros
- Standard API for KVM management
- Works with virsh, virt-manager, cockpit
- Storage pool management
- Virtual network management (NAT, bridge, macvtap)
- Base layer for OpenStack Nova

### Cons
- Single-host management only (no clustering)
- XML-based configuration verbose
- Requires separate tooling for multi-node management
- Learning curve for complex networking

---

## Deployment 9 — Bare Metal → Xen → Guest VMs

### Architecture
```
Physical Hardware
 └── Xen Hypervisor (Type-1, runs directly on metal)
      ├── Dom0 (privileged Linux control domain)
      └── DomU (guest VMs)
           ├── Linux Guest
           ├── Windows Guest
           └── HVM/PV Guests
```

### Use Case
Type-1 hypervisor used in early cloud computing (original AWS), hosting providers, and security-sensitive environments. Still used where strong VM isolation and paravirtualization performance are needed.

### Pros
- True Type-1 hypervisor (Dom0 privilege separation)
- Strong isolation between guests
- Mature — 20+ years of production use
- Paravirtualization (PV) offers excellent I/O performance
- Used historically by AWS

### Cons
- Dom0 is a potential single point of failure
- More complex architecture than KVM
- Requires Xen-aware kernels for PV guests
- Declining adoption versus KVM
- Less community tooling than KVM ecosystem

---

## Deployment 10 — Bare Metal → VMware ESXi → VMs

### Architecture
```
Physical Hardware
 └── VMware ESXi (Type-1 Hypervisor)
      ├── VMFS Datastore
      └── Guest VMs
           ├── Windows Server
           ├── Linux VMs
           └── vSphere-managed cluster (via vCenter)
```

### Use Case
Dominant enterprise virtualization platform. Used in corporate data centers, financial institutions, hospitals, and government — anywhere mature enterprise support, tooling, and Windows workloads are critical.

### Pros
- Most mature enterprise feature set
- Excellent Windows workload support
- vMotion (live migration), HA, DRS clustering
- Rich ecosystem (NSX, vSAN, Tanzu)
- Enterprise support and compliance certifications

### Cons
- Expensive licensing (especially post-Broadcom acquisition)
- Vendor lock-in
- Closed source
- Broadcom acquisition has raised prices significantly
- Requires VMware-certified hardware for full support

---

## Deployment 11 — Bare Metal → Microsoft Hyper-V → VMs

### Architecture
```
Physical Hardware
 └── Hyper-V Hypervisor (Type-1)
      ├── Parent Partition (Windows Server / Windows 10+)
      └── Child VMs
           ├── Windows Server VMs
           ├── Linux VMs
           └── (Azure Stack HCI integration)
```

### Use Case
Microsoft-native virtualization for Windows-centric environments. Used in enterprises running Windows Server workloads, Active Directory, SQL Server, and Azure hybrid deployments.

### Pros
- Free with Windows Server
- Native Windows workload support
- Azure hybrid integration (Azure Arc, Azure Stack HCI)
- Good performance on certified hardware
- PowerShell-based automation

### Cons
- Windows-centric — Linux support is secondary
- GUI management requires SCVMM (additional cost)
- Less flexible than KVM for Linux workloads
- Hyper-V on Windows creates management complexity

---

## Deployment 12 — Bare Metal → Proxmox → KVM VMs + LXC Containers

### Architecture
```
Physical Hardware
 └── Proxmox VE (Debian-based hypervisor OS)
      ├── Web UI / REST API
      ├── KVM Virtual Machines
      │    ├── VM A (Windows)
      │    └── VM B (Linux)
      └── LXC Containers
           ├── Container A
           └── Container B
```

### Use Case
Open-source all-in-one hypervisor for SMBs, homelabs, and mid-size data centers. Combines VMs and containers in a single UI. Popular VMware alternative, especially post-Broadcom ESXi pricing changes.

### Pros
- Free and open-source (enterprise support optional)
- Unified management of VMs + containers
- Web UI is excellent
- Built-in clustering, HA, live migration
- Ceph integration for distributed storage
- Active community

### Cons
- Not as enterprise-polished as VMware/Nutanix
- Ceph setup adds significant complexity
- Limited vendor support for guest OSes
- Web UI can feel limited for very large deployments

---

## Deployment 13 — Bare Metal → Oracle VM (Xen) → VMs

### Architecture
```
Physical Hardware
 └── Oracle VM Server (Xen-based)
      ├── Oracle VM Manager (web console)
      └── Guest VMs
           ├── Oracle Linux
           ├── Windows
           └── Other Linux
```

### Use Case
Oracle workload virtualization — running Oracle Database, Oracle Middleware, and Oracle Applications in a supported, licensed environment.

### Pros
- Oracle-supported for Oracle Database workloads
- Xen performance for Oracle workloads
- Oracle licensing optimization features

### Cons
- Heavily Oracle-ecosystem focused
- Limited community outside Oracle shops
- Declining market share
- More expensive Oracle support costs
- Xen base is aging versus KVM

---

## Deployment 14 — Bare Metal → oVirt → KVM VMs

### Architecture
```
Physical Hardware
 └── Linux + KVM
      └── oVirt Management Engine (web portal)
           ├── oVirt Node Hosts
           └── KVM Virtual Machines
                ├── VM A
                └── VM B
```

### Use Case
Community open-source alternative to VMware vSphere. Used by organizations seeking enterprise VM management without VMware licensing costs. Upstream of Red Hat Virtualization.

### Pros
- Free and open-source
- Enterprise-grade VM management
- Live migration, HA, storage management
- API-driven (REST API)
- Large-scale cluster management

### Cons
- Declining adoption (Red Hat Virtualization EOL)
- Complex initial setup
- UI feels dated
- oVirt Engine requires dedicated server
- Not as actively maintained as Proxmox

---

## Deployment 15 — Bare Metal → XCP-ng → Xen → VMs

### Architecture
```
Physical Hardware
 └── XCP-ng (Xen-based open-source hypervisor)
      ├── Xen Orchestra (web management)
      └── Guest VMs
           ├── Windows VMs
           ├── Linux VMs
           └── XenServer-compatible guests
```

### Use Case
Open-source Citrix XenServer alternative. Community-driven enterprise Xen platform for organizations migrating away from Citrix licensing. Popular in European SMB market.

### Pros
- Free open-source Xen platform
- Xen Orchestra provides excellent UI
- Compatible with Citrix XenServer tooling
- Active community (Vates company backing)
- Good Windows VM support

### Cons
- Xen architecture (Dom0 complexity)
- Smaller ecosystem than KVM-based platforms
- Less tooling integration than Proxmox
- Xen is declining vs KVM long-term

---

## Deployment 16 — Bare Metal → bhyve (FreeBSD) → VMs

### Architecture
```
Physical Hardware
 └── FreeBSD Kernel
      └── bhyve (type-2 hypervisor)
           ├── VM A (Linux guest)
           ├── VM B (Windows guest)
           └── VM N
```

### Use Case
BSD-native hypervisor for running Linux and Windows VMs on FreeBSD. Used in BSD-based hosting providers (notably iXsystems TrueNAS/TrueNAS SCALE), pfSense/OPNsense environments, and FreeBSD development shops.

### Pros
- Native FreeBSD hypervisor
- Benefits from FreeBSD's ZFS + networking stack
- Low overhead
- Used in TrueNAS SCALE for NAS + VMs

### Cons
- FreeBSD ecosystem only
- Less mature than KVM
- Windows support requires additional drivers
- Smaller community than KVM/Xen
- Less tooling for large-scale management

---

## Deployment 17 — Bare Metal → OpenBSD VMM → VMs

*(Covered in Deployment 5 as primary focus; extended here for hypervisor context)*

### Architecture
```
Physical Hardware
 └── OpenBSD Kernel
      └── vmm(4) + vmd(8)
           ├── Linux VM
           └── OpenBSD VM
```

### Use Case
Security-hardened hypervisor on OpenBSD for running isolated Linux services behind OpenBSD's firewall (pf). Used in high-security network infrastructure.

### Pros
- OpenBSD security by default
- Minimal codebase — highly auditable
- Integrates with pf firewall and pledge/unveil

### Cons
- Limited guest OS support
- No Windows guests
- No live migration
- Very limited management tooling
- Not suitable for large-scale deployments

---

## Deployment 18 — Bare Metal → QEMU (Software Emulation, No KVM)

### Architecture
```
Physical Hardware (any architecture)
 └── Linux / macOS / Windows Host OS
      └── QEMU (pure software emulation)
           ├── ARM VM on x86 host
           ├── RISC-V VM on x86 host
           └── Any-arch Guest
```

### Use Case
Cross-architecture emulation for development, testing, and embedded firmware work. Used by embedded developers, kernel developers, and CI pipelines testing on non-native architectures.

### Pros
- Full architecture emulation (ARM, RISC-V, MIPS, etc.)
- No hardware virtualization required
- Excellent for cross-arch testing and development
- Widely used in CI/CD for ARM builds on x86

### Cons
- Very slow — 10-100x slower than native
- Not suitable for production workloads
- High CPU overhead on host
- Complex command-line interface

---

## Deployment 19 — IBM Z Hardware → z/VM → Linux/zOS Guests

### Architecture
```
IBM Z Mainframe Hardware
 └── z/VM Hypervisor
      ├── z/OS Guests (traditional mainframe workloads)
      ├── Linux on Z Guests (thousands per system)
      └── z/VSE Guests
```

### Use Case
Enterprise mainframe virtualization for banking, insurance, government, and healthcare — where extreme reliability, RAS (Reliability, Availability, Serviceability), and thousands of VMs per system are required.

### Pros
- Runs thousands of VMs on single mainframe
- 99.9999% uptime (six nines)
- Unmatched I/O throughput for transaction processing
- Hardware memory encryption and secure boot
- 50+ years of production refinement

### Cons
- Extremely expensive hardware and licensing
- IBM ecosystem lock-in
- Requires specialized mainframe expertise
- Not suitable for general-purpose cloud workloads
- Limited modern cloud-native tooling

---

## Deployment 20 — IBM Power Systems → PowerVM → AIX/Linux/IBM i Guests

### Architecture
```
IBM Power Server (POWER9/POWER10)
 └── PowerVM Hypervisor
      ├── AIX LPARs (logical partitions)
      ├── IBM i LPARs
      └── Linux LPARs
```

### Use Case
Mission-critical enterprise workloads on IBM POWER architecture — ERP systems (SAP), databases (Oracle, DB2), and Unix workloads that require POWER-specific performance.

### Pros
- Excellent for SAP HANA and Oracle workloads
- POWER architecture superior memory bandwidth
- Strong RAS features
- AIX and IBM i native support

### Cons
- Expensive IBM POWER hardware
- IBM ecosystem lock-in
- AIX/IBM i expertise is scarce
- Not cloud-native
- Limited modern container tooling

---

## Deployment 21 — Embedded Hardware → QNX Hypervisor → RTOS + Linux Guests

### Architecture
```
Embedded Hardware (automotive/industrial SoC)
 └── QNX Hypervisor
      ├── QNX RTOS VM (real-time safety functions)
      │    └── Safety-critical systems (braking, steering)
      └── Linux/Android VM (infotainment, connectivity)
```

### Use Case
Automotive systems (instrument clusters, ADAS), industrial automation, and medical devices requiring mixed-criticality — real-time safety functions alongside general-purpose Linux/Android.

### Pros
- Hard real-time guarantees for safety VM
- Proven in automotive (Tier 1 OEM certified)
- Strong safety certifications (ISO 26262, IEC 61508)
- Temporal and spatial isolation between VMs

### Cons
- Expensive QNX licensing
- Specialized expertise required
- Not suitable for general IT/cloud use
- Closed ecosystem

---

## Deployment 22 — Embedded Hardware → INTEGRITY Multivisor

### Architecture
```
Safety-Critical Hardware (aerospace/defense/medical)
 └── INTEGRITY Multivisor (Green Hills Software)
      ├── INTEGRITY RTOS Partition (safety-critical)
      └── Linux/Android Partition (general purpose)
```

### Use Case
Aerospace (DO-178C), defense, and medical devices (IEC 62443) where the highest safety certifications are legally required.

### Pros
- Highest safety certifications available (DO-178C Level A)
- Used in commercial aircraft and military systems
- Provably secure partitioning

### Cons
- Extremely expensive
- Very long certification cycles
- Niche expertise required
- Closed source
- Not relevant outside safety-critical industries

---

## Deployment 23 — Embedded Hardware → PikeOS Hypervisor

### Architecture
```
Embedded Hardware (rail/avionics/automotive)
 └── PikeOS (SYSGO)
      ├── Real-time Partition
      └── Linux Guest Partition
```

### Use Case
Rail signaling, avionics, and defense systems requiring European safety certifications (EN 50128, DO-178C).

### Pros
- European alternative to QNX/INTEGRITY
- Multiple safety certifications
- Deterministic scheduling

### Cons
- Niche market only
- Closed source
- Specialized knowledge required

---

## Deployment 24 — Bare Metal → ACRN Hypervisor → Service VM + Real-time VMs

### Architecture
```
Edge Hardware (Intel x86 IoT/automotive)
 └── ACRN Hypervisor (Linux Foundation project)
      ├── Service VM (Ubuntu/Clear Linux)
      └── Real-time VMs
           ├── RT Linux Guest
           └── IoT workload Guest
```

### Use Case
IoT edge computing, connected vehicles, industrial PCs. Designed for Intel-based embedded systems needing mixed-criticality with open-source tooling.

### Pros
- Open source (Linux Foundation)
- Designed for Intel edge hardware
- Mixed-criticality support
- Smaller footprint than server hypervisors
- Active automotive industry participation

### Cons
- Intel x86 only
- Smaller community than KVM
- Edge/IoT specific — not general-purpose
- Less mature than QNX

---

## Deployment 25 — Bare Metal → Nutanix AHV → VMs + Storage

### Architecture
```
Nutanix-Certified Hardware Nodes
 └── Nutanix AOS (storage fabric)
      └── AHV Hypervisor (KVM-based)
           ├── VMs managed via Prism Central
           ├── Nutanix Files (NAS)
           └── Nutanix Objects (S3)
```

### Use Case
Hyper-converged infrastructure for enterprises seeking a VMware alternative with integrated compute, storage, and networking. Popular in healthcare, education, and distributed enterprise.

### Pros
- Integrated HCI (no separate SAN)
- Excellent management via Prism Central
- Self-healing distributed storage
- Good VMware migration path
- Strong enterprise support

### Cons
- Expensive Nutanix licensing
- Hardware limitations (Nutanix-certified nodes)
- Vendor lock-in
- AHV less feature-rich than ESXi historically

---

## Deployment 26 — Bare Metal → HPE VM Essentials → VMs

### Architecture
```
HPE ProLiant / Synergy Hardware
 └── HPE VM Essentials (KVM-based)
      └── VMs managed via web console
```

### Use Case
VMware replacement targeting HPE hardware customers. Introduced as HPE's response to Broadcom's ESXi price hikes.

### Pros
- Lower cost than VMware post-Broadcom
- Tight integration with HPE hardware (iLO, OneView)
- Familiar VM management

### Cons
- HPE hardware lock-in
- Newer product — less proven at scale
- Smaller ecosystem than VMware

---

## Deployment 27 — Bare Metal → Huawei FusionCompute → VMs

### Architecture
```
Huawei Hardware (TaiShan/FusionServer)
 └── FusionCompute (KVM-based hypervisor)
      ├── FusionManager (management)
      └── VMs
```

### Use Case
Enterprise virtualization in Chinese domestic markets and Huawei-ecosystem deployments. Used in Chinese banks, telecoms, and government.

### Pros
- Deep Huawei hardware integration
- Cost-effective in Chinese market
- Supports ARM (Kunpeng) + x86 nodes

### Cons
- Geopolitical concerns in Western markets
- Smaller ecosystem outside China
- English documentation limited
- Vendor lock-in to Huawei hardware

---

## Deployment 28 — Bare Metal → Scale Computing HC3 → VMs

### Architecture
```
Scale Computing Hardware Nodes
 └── HyperCore OS (KVM-based HCI)
      └── VMs
           ├── Edge workloads
           └── Remote office VMs
```

### Use Case
Edge computing and remote office/branch office (ROBO) virtualization for organizations needing simple, self-healing HCI without dedicated IT staff.

### Pros
- Very simple management (designed for non-experts)
- Self-healing storage
- Low cost for edge deployments
- Good for retail, manufacturing, distributed offices

### Cons
- Limited scale vs Nutanix/VMware
- Closed hardware platform
- Not suitable for large data center deployments

---

# CATEGORY 3: Container Orchestration on Bare Metal

---

## Deployment 29 — Bare Metal → Kubernetes → Containers

### Architecture
```
Physical Hardware Nodes
 └── Linux OS (Ubuntu/RHEL/Flatcar)
      └── Kubernetes
           ├── Control Plane (API server, etcd, scheduler)
           ├── Worker Nodes
           └── Pods (containers via containerd/CRI-O)
                ├── App Pod A
                ├── App Pod B
                └── System Pods (DNS, CNI, CSI)
```

### Use Case
Cloud-native application deployment on bare metal for organizations wanting maximum performance without VM overhead. Used by financial services, telcos, and tech companies running stateless microservices.

### Pros
- No VM overhead — maximum container density
- Declarative GitOps-friendly management
- Massive ecosystem (Helm, operators, CNCF tools)
- Auto-scaling, self-healing workloads
- Multi-cloud portable

### Cons
- No VM workload support (needs KubeVirt)
- Bare metal node provisioning is complex
- etcd requires careful maintenance
- Networking complexity (CNI plugins)
- Steep learning curve

---

## Deployment 30 — Bare Metal → k3s → Containers

### Architecture
```
Physical Hardware (Edge / Low-resource node)
 └── Linux OS
      └── k3s (lightweight Kubernetes)
           ├── Single-binary control plane
           └── Worker node Pods
```

### Use Case
Edge computing, IoT gateways, Raspberry Pi clusters, and resource-constrained environments needing Kubernetes without the full resource overhead.

### Pros
- Single binary — ~70MB
- Works on ARM and x86
- Auto-configured SQLite (no external etcd needed for small clusters)
- Full Kubernetes API compatibility
- Fast startup and low memory footprint

### Cons
- Some enterprise Kubernetes features absent
- SQLite backend not suited for large HA clusters
- Not recommended for very large-scale production
- Less tested at hyperscale

---

## Deployment 31 — Bare Metal → Docker Swarm → Containers

### Architecture
```
Physical Hardware Nodes
 └── Linux OS
      └── Docker Engine
           └── Docker Swarm (cluster mode)
                ├── Manager Nodes
                └── Worker Nodes
                     └── Service Containers
```

### Use Case
Simple container orchestration for small to medium teams already using Docker. Used when Kubernetes complexity is unnecessary — internal tools, small microservices, and development teams wanting simple multi-node deploys.

### Pros
- Very simple setup — built into Docker
- Easy to understand and operate
- Good for small clusters
- Familiar Docker Compose-like syntax (stack files)

### Cons
- Docker Inc declining investment in Swarm
- Limited compared to Kubernetes (no RBAC, limited scheduling)
- No Helm/operator ecosystem
- Not recommended for large-scale production
- Effectively in maintenance mode

---

## Deployment 32 — Bare Metal → Nomad → Containers + QEMU VMs

### Architecture
```
Physical Hardware
 └── Linux OS
      └── HashiCorp Nomad
           ├── Container workloads (Docker/containerd)
           ├── QEMU VM workloads
           ├── Raw binary workloads
           └── Java workloads
```

### Use Case
Organizations wanting a simpler orchestrator than Kubernetes that handles mixed workloads (containers, VMs, binaries). Used in HashiCorp-ecosystem shops, fintech, and environments needing heterogeneous workload scheduling.

### Pros
- Much simpler than Kubernetes
- Multi-workload: containers, VMs, binaries, Java
- Integrates natively with Consul and Vault
- Low operational overhead
- Fast job scheduling

### Cons
- Smaller ecosystem than Kubernetes
- No built-in service mesh (needs Consul)
- Less community support and fewer operators
- Not as cloud-native as K8s for container-first workloads

---

## Deployment 33 — Bare Metal → Linux → Apptainer (HPC Containers)

### Architecture
```
HPC Cluster (bare metal nodes)
 └── Linux OS (no root daemon)
      └── Apptainer (formerly Singularity)
           └── HPC Container Images
                ├── Scientific simulation
                ├── ML training job
                └── Genomics pipeline
```

### Use Case
Scientific computing, national labs, universities, and research clusters where Docker's root daemon is a security concern and MPI-parallel workloads need container portability.

### Pros
- No root daemon — rootless by design
- MPI and GPU support built-in
- Runs on shared HPC clusters (SLURM-compatible)
- Reproducible scientific environments
- Can convert Docker images

### Cons
- Not designed for long-running services
- No orchestration layer (use SLURM)
- Network isolation less sophisticated than Docker
- Not suitable for microservices architecture

---

---

# CATEGORY 14: Switched / Reversed Control Plane Models

---

## Deployment 81 — Bare Metal → LXD Cluster → VMs + Containers (Unified)

### Architecture
```
Physical Hardware (multiple nodes)
 └── LXD Cluster (peer cluster via lxd init --auto)
      ├── LXC System Containers
      │    ├── App containers
      │    └── Service containers
      └── QEMU VMs (managed by LXD)
           ├── Windows VM
           └── Legacy Linux VM
```

### Use Case
Unified container and VM management without Kubernetes complexity. Hosters, ISPs, and SMBs using Canonical's stack for a simple all-in-one virtualization platform.

### Pros
- Single tool for both containers and VMs
- Clustering included
- REST API for automation
- Snapshots, migrations, profiles

### Cons
- Not cloud-native (no K8s ecosystem)
- LXD/Incus community split after Canonical fork
- Limited at hyperscale

---

## Deployment 82 — Bare Metal → KVM → VM → LXD Cluster → Containers

### Architecture
```
Physical Hardware
 └── KVM
      └── VMs (LXD host nodes)
           └── LXD Cluster (across VMs)
                └── LXC Containers
                     └── Tenant workloads
```

### Use Case
Virtualizing the container infrastructure — isolating LXD clusters within VMs for multi-tenant or DR scenarios.

### Pros
- VM isolation for LXD clusters
- Good for multi-tenant container hosting
- Easy LXD cluster recovery (redeploy VM)

### Cons
- Double overhead
- Nested cgroup complexity
- More complex than direct LXD on metal

---

## Deployment 83 — Bare Metal → Kubernetes → Metal3 → Installs OpenStack on New Nodes

### Architecture
```
Physical Hardware Pool
 └── Kubernetes (management cluster)
      └── Metal3 (provisions new bare metal nodes)
           └── Provisioned Nodes
                └── OpenStack installation
```

### Use Case
GitOps-driven OpenStack deployment where Kubernetes and Metal3 manage the hardware lifecycle, and OpenStack is deployed on top.

### Pros
- Fully GitOps-driven cloud provisioning
- Self-healing node replacement
- Declarative bare metal management

### Cons
- Extremely complex
- Multiple tools to master
- Limited production examples

---

## Deployment 84 — Bare Metal → OpenStack Neutron Only + Kubernetes

### Architecture
```
Physical Hardware
 └── OpenStack Neutron (SDN only, no compute)
      └── Kubernetes cluster
           └── CNI uses Neutron networks
```

### Use Case
Organizations wanting OpenStack's advanced SDN (Neutron) alongside Kubernetes compute. Leverages existing Neutron investment while migrating to K8s.

### Pros
- Advanced network policy from Neutron
- Leverage existing Neutron investment

### Cons
- Unusual architecture — limited tooling
- Two systems for one concern
- Most organizations choose pure CNI instead

---

## Deployment 85 — Bare Metal → Ironic Standalone + Kubernetes (No OpenStack Nova)

### Architecture
```
Physical Hardware
 └── Ironic (standalone provisioner)
      └── Provisions bare metal nodes
           └── Kubernetes (primary control plane)
```

### Use Case
Organizations needing hardware provisioning from Ironic but using Kubernetes for everything else — eliminating full OpenStack complexity.

### Pros
- Minimal OpenStack footprint
- Kubernetes for all workload management
- Still automated hardware provisioning

### Cons
- Ironic standalone is less documented
- Must replace Nova/Neutron/Cinder with K8s-native solutions

---

# CATEGORY 15: Research / Academic / Specialized

---

## Deployment 86 — Bare Metal → seL4 Microkernel → Isolated VMs

### Architecture
```
Physical Hardware
 └── seL4 Microkernel (formally verified)
      ├── Isolated VM A (safety-critical)
      └── Isolated VM B (general purpose)
```

### Use Case
High-assurance security: defense, critical infrastructure, avionics — where the hypervisor must be formally mathematically verified.

### Pros
- Formally verified (mathematically proven correct)
- Highest assurance security available
- DARPA-funded research, used in classified projects

### Cons
- Extremely niche expertise
- Very limited application ecosystem
- Not for general IT use

---

## Deployment 87 — Bare Metal → Linux → Jailhouse → Static Real-Time VMs

### Architecture
```
Physical Hardware
 └── Linux OS (inmate manager)
      └── Jailhouse (partitioning hypervisor)
           ├── Root cell (Linux continues running)
           └── Inmate cells (real-time partitions)
                ├── RTOS inmate
                └── Bare-metal application
```

### Use Case
Industrial control systems and robotics where Linux runs alongside hardware-isolated real-time partitions.

### Pros
- Non-destructive (Linux keeps running)
- Hard real-time guarantee in inmate cells
- Open source (GPL)

### Cons
- Static partitioning — not dynamic
- Limited hardware support
- Small community

---

## Deployment 88 — Bare Metal → Barrelfish (Research OS)

### Architecture
```
Multi-Core Hardware
 └── Barrelfish OS
      └── Per-core OS instances (heterogeneous)
           └── Message-passing between cores
```

### Use Case
Academic research into heterogeneous multi-core OS architectures. University research only.

### Pros
- Novel multi-core isolation research
- Heterogeneous CPU support

### Cons
- Research only — not production
- No ecosystem

---

## Deployment 89 — Bare Metal → NOVA Microhypervisor

### Architecture
```
Physical Hardware
 └── NOVA Microhypervisor (TU Dresden)
      └── Guest VMs with minimal TCB
```

### Use Case
Security research minimizing the Trusted Computing Base (TCB) of the hypervisor.

### Pros
- Minimal TCB
- Good for security research

### Cons
- Research project, limited production use

---

## Deployment 90 — Bare Metal → Xen Research Forks → Experimental VMs

### Architecture
```
Physical Hardware
 └── Xen Research Fork
      └── Experimental VMs with custom isolation
```

### Use Case
Academic hypervisor security research — VM introspection, memory deduplication, novel isolation mechanisms.

### Pros
- Well-documented codebase for research
- LibVMI and research tools available

### Cons
- Not production-ready
- Requires hypervisor research expertise

---

# CATEGORY 16: Immutable OS + Kubernetes

---

## Deployment 91 — Bare Metal → Talos Linux → Kubernetes → Containers

*(Primary focus in Deployment 43, containers-only variant here)*

### Architecture
```
Physical Hardware
 └── Talos Linux (immutable, API-only OS)
      └── Kubernetes
           └── Container workloads
```

### Use Case
Security-first Kubernetes — no shell access, API-only management, CIS compliant by default.

### Pros / Cons — See Deployment 43

---

## Deployment 92 — Bare Metal → Flatcar Container Linux → Kubernetes

### Architecture
```
Physical Hardware
 └── Flatcar Container Linux (CoreOS successor)
      └── systemd + containerd
           └── Kubernetes
                └── Container workloads
```

### Use Case
Auto-updating immutable Linux for large Kubernetes node fleets. Microsoft-backed, active development.

### Pros
- Automatic atomic OS updates (A/B partitions)
- Container-optimized — no package manager
- Minimal attack surface
- Large node fleet management

### Cons
- Cannot install arbitrary packages
- Different operational model
- Microsoft backing creates some community concern

---

## Deployment 93 — Bare Metal → k3OS → Edge Kubernetes

### Architecture
```
Edge Hardware
 └── k3OS (OS + k3s bundled)
      └── k3s Kubernetes
           └── Edge workloads
```

### Use Case
Lightweight edge where OS and Kubernetes bootstrap together.

### Pros
- Simple single-binary deployment
- Low resources

### Cons
- Limited development momentum
- Merged into Harvester roadmap

---

## Deployment 94 — Bare Metal → NixOS → KVM → VMs

### Architecture
```
Physical Hardware
 └── NixOS (declarative, reproducible)
      └── KVM (via nixpkgs)
           └── Declaratively-defined VMs
```

### Use Case
Fully reproducible, version-controlled virtualization infrastructure.

### Pros
- 100% reproducible deployments
- Atomic OS rollbacks
- Declarative VM definitions

### Cons
- Steep Nix learning curve
- Limited enterprise support

---

## Deployment 95 — Bare Metal → RancherOS (Deprecated) → Kubernetes

### Architecture
```
Physical Hardware
 └── RancherOS (deprecated)
      └── Kubernetes via Rancher
```

### Use Case
Historical — deprecated Docker-first OS. **Do not use for new deployments.**

### Pros
- Simple container-first OS (historically)

### Cons
- Deprecated — replaced by k3OS, Harvester, Flatcar

---

# CATEGORY 17: Enterprise HCI

---

## Deployment 96 — Bare Metal → Nutanix AHV → VMs + Integrated Storage

*(Extended from Deployment 25)*

### Architecture
```
Nutanix-Certified Nodes
 └── Nutanix AOS (distributed storage)
      └── AHV Hypervisor (KVM-based)
           ├── Prism Central (management)
           ├── Nutanix Files / Objects / Volumes
           └── Guest VMs
```

### Use Case
Enterprise HCI replacing SAN + VMware. Key: VDI, databases, branch offices.

### Pros
- Best-in-class HCI management (Prism Central)
- Hardware-agnostic (Dell, HPE, Lenovo)
- 1-click upgrades, self-healing storage
- Strong VDI and database performance

### Cons
- Expensive licensing
- AHV still catching up to ESXi in some features
- Nutanix software lock-in

---

## Deployment 97 — Bare Metal → VMware ESXi → vCenter + Tanzu (K8s on VMware)

### Architecture
```
VMware Hardware
 └── VMware ESXi
      └── vCenter
           ├── Traditional VMs
           └── vSphere with Tanzu
                └── Kubernetes Workload Clusters
                     └── Pods + KubeVirt VMs
```

### Use Case
Enterprises with VMware investments adding Kubernetes without replacing infrastructure.

### Pros
- Leverage existing VMware skills
- vSAN, NSX integration
- Enterprise support

### Cons
- Very expensive post-Broadcom
- Vendor lock-in
- Complex license structure

---

## Deployment 98 — Bare Metal → Hyper-V → Azure Stack HCI

### Architecture
```
Azure Stack HCI Certified Hardware
 └── Hyper-V
      └── Azure Stack HCI OS
           ├── Storage Spaces Direct
           ├── VMs
           └── Azure Arc integration
```

### Use Case
Microsoft-centric enterprises wanting Azure services on-premises with AKS on-prem.

### Pros
- Azure Arc hybrid management
- AKS on Azure Stack HCI
- Azure Backup/Monitor integration

### Cons
- Requires Azure subscription
- Windows-centric
- Microsoft cloud connectivity dependency

---

## Deployment 99 — Bare Metal → Proxmox → Ceph + KVM + LXC (Full HCI)

### Architecture
```
Physical Hardware (3+ nodes)
 └── Proxmox VE
      ├── Ceph (distributed storage)
      ├── KVM Virtual Machines
      └── LXC Containers
```

### Use Case
Low-cost open-source HCI for SMBs, research labs, and VMware migration.

### Pros
- Zero licensing cost
- Excellent web UI
- Integrated Ceph + VM + container management
- Live migration with Ceph
- Strong VMware migration tools

### Cons
- Ceph adds operational complexity
- Not as polished at enterprise scale as Nutanix/VMware
- Limited enterprise integrations

---

## Deployment 100 — Bare Metal → Red Hat Virtualization (oVirt + KVM)

### Architecture
```
Physical Hardware
 └── Red Hat Virtualization (RHV)
      ├── RHV Manager
      ├── RHV Hosts (KVM)
      └── VMs (RHEL + Windows)
```

### Use Case
Red Hat-supported enterprise VM management. **Note: EOL announced June 2024 — migrate to OpenShift Virtualization.**

### Pros
- Red Hat enterprise support
- Strong RHEL workload support
- Live migration, HA, storage management

### Cons
- End of life — no new features
- Must migrate to OpenShift Virtualization
- Declining investment

---

# CATEGORY 18: Full-Stack Extreme Combinations

---

## Deployment 101 — Near Hyperscaler: Ceph + OpenStack (Ironic + VMs) + Kubernetes (KubeVirt + Metal3)

### Architecture
```
Physical Hardware Pool
 ├── Ceph Cluster (block + object + file)
 ├── OpenStack
 │    ├── Ironic (bare metal provisioning)
 │    ├── Nova + KVM (VM compute)
 │    ├── Neutron (SDN)
 │    └── Cinder/Glance → Ceph
 └── Kubernetes
      ├── KubeVirt (VMs)
      ├── Metal3 (bare metal)
      └── Rook-Ceph (storage)
```

### Use Case
Large telco, national cloud providers, research institutions — maximum workload flexibility: IaaS tenants (OpenStack), cloud-native (K8s), bare metal (HPC), all sharing Ceph.

### Pros
- Maximum workload flexibility
- No vendor lock-in
- Shared storage reduces duplication
- Suitable for very large multi-tenant deployments

### Cons
- Extremely high operational complexity
- Requires large specialized team
- Months-long deployment
- Very high expertise cost

---

## Deployment 102 — Kubernetes → OpenStack Control Plane (Containerized) + KubeVirt + Metal3

### Architecture
```
Physical Hardware
 └── Kubernetes (base)
      ├── OpenStack-Helm / OpenStack Operator (pods)
      │    ├── Nova → KVM on compute nodes
      │    └── Neutron → SDN
      ├── KubeVirt → VM workloads
      └── Metal3 → Bare metal node provisioning
```

### Use Case
Modern GitOps-managed OpenStack where the control plane runs as Kubernetes pods, enabling K8s-native upgrade and recovery.

### Pros
- OpenStack upgrades = Helm upgrades
- Self-healing control plane
- GitOps-friendly
- Single K8s control plane for everything

### Cons
- Extreme expertise requirement
- Complex multi-layer debugging
- Few public case studies

---

## Deployment 103 — OpenNebula + Kubernetes + KubeVirt + Ceph

### Architecture
```
Physical Hardware
 ├── Ceph
 └── OpenNebula
      ├── KVM VMs → Ceph RBD
      └── Kubernetes (provisioned by ONE)
           └── KubeVirt VMs
```

### Use Case
Simpler-than-OpenStack private cloud with modern container + VM support and Ceph storage.

### Pros
- Simpler than OpenStack
- Full VM + container + storage
- Good for research and mid-size enterprise

### Cons
- Non-mainstream combination
- Limited documentation
- Smaller community

---

## Deployment 104 — XCP-ng + Ceph + CloudStack

### Architecture
```
Physical Hardware
 ├── Ceph Cluster
 └── XCP-ng (Xen hypervisor)
      └── CloudStack (management)
           ├── Xen VMs
           └── Ceph RBD backend
```

### Use Case
Fully open-source alternative to VMware + vSAN + vCenter for hosting providers wanting AWS-compatible APIs.

### Pros
- Fully open source, no licensing
- Strong multi-tenancy (CloudStack)
- AWS-compatible API

### Cons
- Xen + CloudStack expertise required
- Ceph + CloudStack integration less mature than with OpenStack

---

## Deployment 105 — Nomad + Firecracker (Serverless MicroVMs)

### Architecture
```
Physical Hardware
 └── HashiCorp Nomad
      └── Firecracker task driver
           ├── Nomad job → Firecracker MicroVM
           └── Fast scheduling + VM isolation per job
```

### Use Case
HashiCorp-stack organizations wanting VM-isolated workloads without Kubernetes complexity.

### Pros
- Simpler than Kubernetes
- VM isolation per workload
- Fast Firecracker startup
- Integrates with Consul + Vault

### Cons
- Nomad Firecracker driver less mature
- Smaller ecosystem than K8s
- Firecracker device limitations

---

## Deployment 106 — StarlingX + Ceph → Edge Cloud

### Architecture
```
Edge Hardware (2+ nodes)
 ├── Ceph (edge distributed storage)
 └── StarlingX
      ├── OpenStack VMs (VNFs)
      └── Kubernetes containers (CNFs)
```

### Use Case
5G MEC and telco edge requiring VMs + containers + persistent storage on minimal hardware.

### Pros
- Carrier-grade HA on 2 nodes
- Both VMs and containers
- Ceph for persistent edge storage

### Cons
- Very complex for edge
- High minimum hardware requirements
- Slow release cadence

---

## Deployment 107 — IBM Z → z/VM + Linux Guests + Ceph (Mainframe Cloud)

### Architecture
```
IBM Z Mainframe
 └── z/VM (thousands of VMs)
      └── Linux on Z Guests
           ├── OpenStack control plane VMs
           ├── Kubernetes worker VMs
           └── Ceph OSD VMs
```

### Use Case
Banks and insurance companies building a modern cloud on IBM Z mainframe infrastructure.

### Pros
- Mainframe reliability (six nines)
- Extremely high VM density
- Modern cloud API on trusted hardware

### Cons
- Extremely expensive
- IBM ecosystem lock-in
- Very specialized expertise required

---

## Deployment 108 — ACRN + OpenStack Edge + StarlingX (Experimental)

### Architecture
```
Industrial/Automotive Edge Hardware
 └── ACRN Hypervisor (real-time base)
      └── Service VM running StarlingX
           └── OpenStack Edge → VMs + containers
```

### Use Case
Next-generation industrial/automotive edge combining ACRN real-time guarantees with cloud management.

### Pros
- Real-time hardware isolation
- Cloud management for non-RT workloads
- Intel edge silicon optimized

### Cons
- Highly experimental
- Intel hardware only
- No public production deployments

---

## Deployment 109 — Kubernetes + OpenFaaS + Firecracker (Full Serverless Stack)

### Architecture
```
Physical Hardware
 └── Kubernetes
      ├── OpenFaaS (function orchestrator)
      │    └── Functions → Firecracker MicroVMs
      └── Prometheus + Grafana (monitoring)
```

### Use Case
Self-hosted AWS Lambda alternative on bare metal for data residency compliance and cost savings.

### Pros
- True serverless on private infrastructure
- Strong Firecracker VM isolation per function
- Cost savings vs AWS Lambda at scale
- Data residency compliance

### Cons
- High operational overhead
- Cold start latency
- Not as managed as AWS Lambda
- Complex multi-layer debugging

---

## Deployment 110 — Xen + KubeVirt + Ceph (Alternative Full Cloud)

### Architecture
```
Physical Hardware
 ├── Ceph Cluster
 └── Xen (XCP-ng or upstream)
      └── Kubernetes (on Xen VMs or Dom0)
           ├── KubeVirt (additional VMs)
           └── Rook-Ceph (storage)
```

### Use Case
Organizations using Xen as base hypervisor wanting to add Kubernetes and cloud-native capabilities without abandoning Xen.

### Pros
- Xen strong base isolation
- Kubernetes cloud-native on top
- Ceph unified storage

### Cons
- KubeVirt normally targets KVM — Xen is non-standard
- Limited testing of this combination
- Xen expertise declining

---

# Summary Reference Table

| # | Deployment | Complexity | Best For | Maturity |
|---|-----------|-----------|---------|---------|
| 1 | Linux → LXC | Low | Embedded, DIY | High |
| 2 | LXD → LXC | Low-Med | VPS, SMB containers | High |
| 3 | systemd-nspawn | Low | Dev/test | Medium |
| 4 | FreeBSD Jails | Low-Med | BSD hosting | Very High |
| 5 | OpenBSD VMM | Low | Security research | Medium |
| 7 | KVM + QEMU | Medium | Universal VMs | Very High |
| 8 | libvirt → KVM | Medium | Managed VM hosts | Very High |
| 9 | Xen | Med-High | Hosting, security | High |
| 10 | VMware ESXi | Medium | Enterprise Windows | Very High |
| 11 | Hyper-V | Medium | Microsoft shops | High |
| 12 | Proxmox | Medium | SMB, VMware migration | High |
| 19 | IBM z/VM | Very High | Banking, gov | Very High |
| 21 | QNX Hypervisor | High | Automotive, medical | High |
| 24 | ACRN | High | IoT, automotive | Medium |
| 29 | K8s Containers | High | Cloud-native apps | Very High |
| 30 | k3s | Medium | Edge, IoT | High |
| 32 | Nomad | Medium | Multi-workload | High |
| 34 | KVM → VMs → K8s | High | Enterprise K8s | Very High |
| 38 | K8s → KubeVirt | High | Unified VM+container | Med-High |
| 39 | K8s → Kata | High | Multi-tenant K8s | Medium |
| 40 | K8s → Firecracker | High | Serverless | High |
| 41 | K8s → Harvester | High | VMware migration | Medium |
| 42 | OpenShift + KubeVirt | Very High | Red Hat enterprise | High |
| 43 | Talos + K8s | High | Security-first K8s | Med-High |
| 46 | OpenStack + KVM | Very High | Private cloud IaaS | High |
| 47 | OpenStack Ironic | High | Bare metal cloud | High |
| 48 | OpenStack Magnum | Very High | K8s-as-a-Service | Medium |
| 51 | MAAS | High | Ubuntu bare metal | High |
| 52 | K8s + Metal3 | High | GitOps bare metal | Medium |
| 55 | CloudStack | High | Hosting providers | High |
| 56 | OpenNebula | Medium-High | Research/EU cloud | High |
| 58 | Ceph + OpenStack | Very High | Large private cloud | High |
| 59 | Ceph + K8s + KubeVirt | Very High | Cloud-native HCI | Medium |
| 62 | Firecracker | High | Serverless, FaaS | High |
| 66 | Xen + MirageOS | Very High | Security research | Low (prod) |
| 70 | OPNFV | Very High | Telco NFV | High |
| 71 | StarlingX | Very High | Telco edge, 5G | Med-High |
| 86 | seL4 | Extreme | Defense, certified | Low (prod) |
| 87 | Jailhouse | High | Industrial RT | Medium |
| 91 | Talos + K8s | High | Security K8s | Med-High |
| 92 | Flatcar + K8s | Medium | Large fleets | High |
| 96 | Nutanix AHV | Medium | Enterprise HCI | Very High |
| 97 | VMware + Tanzu | High | VMware K8s | Very High |
| 98 | Hyper-V + Azure HCI | High | Microsoft hybrid | High |
| 99 | Proxmox + Ceph HCI | High | SMB HCI | High |
| 100 | Red Hat Virt (EOL) | High | Legacy RHV | N/A (EOL) |
| 101 | Full Hyperscaler Stack | Extreme | Telco, national cloud | High (components) |
| 105 | Nomad + Firecracker | High | HashiCorp serverless | Medium |
| 109 | K8s + OpenFaaS + Firecracker | High | Private serverless | Medium |

---

*Document covers 110 unique bare-metal virtualization deployment architectures across 18 categories.*  
*Generated: February 2026*
