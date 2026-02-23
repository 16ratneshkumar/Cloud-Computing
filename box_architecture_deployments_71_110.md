# Complete Box-Type Architecture Diagrams
## Deployments 71–110 | Full Stack with Use Cases, Pros & Cons

---

# DEPLOYMENT 71 — Bare Metal → StarlingX → Containers + VMs (Telco Edge)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TELCO EDGE WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CNFs (Cloud-Native Network Functions)                   │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │ 5G gNB-CU    │  │  UPF (User   │  │  vRouter     │   │   │
│  │  │ (Kubernetes  │  │  Plane Func) │  │  (container) │   │   │
│  │  │  pod)        │  │  (DPDK pod)  │  │              │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  │  VNFs (VM-based Network Functions)                       │   │
│  │  ┌──────────────┐  ┌──────────────────────────────────┐  │   │
│  │  │ Legacy vEPC  │  │  vBBU (Baseband Unit)            │  │   │
│  │  │ (KVM VM)     │  │  (KVM VM with CPU pinning)       │  │   │
│  │  └──────────────┘  └──────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              STARLINGX PLATFORM                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  StarlingX System Controller (central management)        │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  System Management (sysinv) — hardware inventory   │  │   │
│  │  │  Fault Management (fm) — alarm, log, fault mgmt    │  │   │
│  │  │  Software Management (usm) — patch orchestration   │  │   │
│  │  │  Configuration Management — host config & services │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KUBERNETES (embedded, StarlingX-tuned)                  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Multus CNI │ SR-IOV device plugin │ CPU Manager   │  │   │
│  │  │  NUMA-aware scheduling │ HugePages management      │  │   │
│  │  │  Node Feature Discovery (NFD)                      │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OPENSTACK (optional, for VM-based VNFs)                 │   │
│  │  Nova + KVM │ Neutron + OVS/DPDK │ Cinder storage       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              PERFORMANCE-TUNED HOST OS (CentOS/Debian based)    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  DPDK (user-space network I/O bypass)                    │   │
│  │  HugePages (2MB/1GB pages for DPDK + VNF memory)        │   │
│  │  CPU isolation (isolcpus for RT + DPDK cores)            │   │
│  │  PREEMPT_RT kernel patch (real-time Linux)               │   │
│  │  IOMMU (VT-d for SR-IOV NIC passthrough)                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (COTS Edge Servers)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CONFIGURATION OPTIONS:                                  │   │
│  │                                                          │   │
│  │  All-in-One Simplex (AIO-SX):    1 server total         │   │
│  │  CPU 16c+ │ RAM 64GB+ │ NVMe │ SR-IOV NIC 25GbE         │   │
│  │  (dev/test only — no HA)                                 │   │
│  │                                                          │   │
│  │  All-in-One Duplex (AIO-DX):     2 servers              │   │
│  │  (control plane + workload HA — 2-node minimum)         │   │
│  │                                                          │   │
│  │  Standard (3+ nodes):                                   │   │
│  │  Controller×2 │ Worker×N │ Storage×N                   │   │
│  │  CPU (Intel Xeon D/SP) │ RAM 256GB+ │ 25GbE×2 │ NVMe   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

MULTI-SITE TOPOLOGY:
┌───────────────────────────────────────────────────────────────┐
│  Central Cloud (data center)                                  │
│  StarlingX System Controller                                  │
│  ┌──────────────────┐  ┌──────────────────────────────────┐  │
│  │  Subcloud 1      │  │  Subcloud 2                      │  │
│  │  (edge site 1)   │  │  (edge site 2)                   │  │
│  │  AIO-DX          │  │  AIO-DX                          │  │
│  └──────────────────┘  └──────────────────────────────────┘  │
│  (managed via Distributed Cloud - dcmanager)                  │
└───────────────────────────────────────────────────────────────┘
```

## How It Works
- StarlingX installs on bare metal COTS servers with a heavily tuned OS for real-time and high-throughput workloads
- Kubernetes is embedded and tuned for NFV (SR-IOV, DPDK, NUMA, HugePages, CPU pinning)
- OpenStack can optionally be included for VM-based VNFs
- StarlingX's Distributed Cloud (dcmanager) manages central + remote edge subclouds from one controller
- PREEMPT_RT kernel provides real-time guarantees for time-critical network functions

## Use Case
Telco 5G edge, central office virtualization, and distributed cloud for mobile operators. Used by Wind River (commercial) and OpenInfra Foundation (open source).

## Pros
- Purpose-built for telco NFV and edge
- PREEMPT_RT for real-time network functions
- Integrated DPDK, SR-IOV, HugePages management
- 2-node HA (AIO-DX) — lowest footprint HA deployment
- Distributed cloud management at scale

## Cons
- Extremely complex — combines OpenStack + K8s + custom telco services
- Telco-specific knowledge required
- Long deployment times
- Heavy hardware requirements even for edge (64GB RAM minimum)

---

# DEPLOYMENT 72 — Bare Metal → Kubernetes → OSM MANO → CNFs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              NETWORK OPERATOR INTERFACES                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OSM UI (web) │ osmclient CLI │ REST API │ Northbound API │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OSM (Open Source MANO) — ETSI MANO reference impl  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OSM Core (runs as Docker Compose or Kubernetes pods)    │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  NBI (North Bound Interface) — REST API           │  │   │
│  │  │  LCM (Life Cycle Manager) — VNF/NS instantiation  │  │   │
│  │  │  RO (Resource Orchestrator) — vim account mgmt    │  │   │
│  │  │  MON (Monitoring) — VNF metrics collection        │  │   │
│  │  │  POL (Policy) — auto-scaling / auto-healing        │  │   │
│  │  │  SOL005 (ETSI NFV API compliance)                  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Kafka (event bus between OSM modules)                   │   │
│  │  MongoDB (VNF/NS descriptor store)                       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  VIM (Virtualized Infrastructure Manager)
┌────────────────────────────▼────────────────────────────────────┐
│              VIM LAYER (one or many)                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes VIM          │  OpenStack VIM                │   │
│  │  (K8s cluster for CNFs)  │  (for VM-based VNFs)         │   │
│  │  ┌────────────────────┐  │  ┌────────────────────────┐  │   │
│  │  │  CNF Pods          │  │  │  VNF VMs               │  │   │
│  │  │  (free5GC 5G core) │  │  │  (vIMS, vEPC)          │  │   │
│  │  │  (Calico / Multus) │  │  │  (Nova + KVM)          │  │   │
│  │  └────────────────────┘  │  └────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + KVM (compute nodes)                 │
│  DPDK │ SR-IOV │ HugePages │ CPU pinning                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (telco COTS)                   │
│  Intel Xeon │ RAM 256GB+ │ SR-IOV NICs │ 25GbE+ NIC            │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- OSM implements the ETSI NFV MANO architecture: NFV Orchestrator + VNF Manager + VIM
- NBI receives service deployment requests (NS descriptors, VNF descriptors in ETSI format)
- LCM orchestrates VNF instantiation across VIMs (Kubernetes for CNFs, OpenStack for VNFs)
- MON collects metrics; POL applies autoscaling policies based on metrics
- OSM manages multi-site, multi-VIM deployments from a single control point

## Use Case
European telco operators (Telecom Italia, Telefonica) deploying open-source MANO for network service orchestration across multiple sites and VIMs.

## Pros
- ETSI-compliant open source MANO
- Supports K8s + OpenStack VIMs simultaneously
- Active ETSI-led community
- Multi-VIM, multi-site orchestration

## Cons
- ETSI MANO model complex to learn
- OSM has had stability issues at scale
- Slow pace of innovation vs K8s-native tools
- Heavy operational overhead

---

# DEPLOYMENT 73 — Bare Metal → OpenStack + Kubernetes → ONAP → VNFs/CNFs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TELECOM SERVICE DESIGN & ACTIVATION                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ONAP Portal │ SDC (Service Design Center) │ CLI         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              ONAP (Open Network Automation Platform)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Runs entirely as Kubernetes pods (Helm charts)          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  SDC   — Service Design Center (model services)    │  │   │
│  │  │  SO    — Service Orchestrator (instantiate)        │  │   │
│  │  │  A&AI  — Active & Available Inventory             │  │   │
│  │  │  VID   — Virtual Infrastructure Deployment        │  │   │
│  │  │  SDNC  — SDN Controller (OpenDaylight)             │  │   │
│  │  │  DCAE  — Data Collection, Analysis, Events        │  │   │
│  │  │  Policy — Policy Engine (Drools)                   │  │   │
│  │  │  CDS   — Controller Design Studio                 │  │   │
│  │  │  CLAMP — Closed Loop Automation Management         │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Supporting: Cassandra │ Kafka │ MariaDB │ Elasticsearch │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  ONAP calls VIM APIs
┌────────────────────────────▼────────────────────────────────────┐
│              VIM LAYER                                          │
│  OpenStack (VMs for VNFs) │ Kubernetes (pods for CNFs)         │
│  K8s clusters provisioned via SO → SO-CNF adapter              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              INFRASTRUCTURE                                     │
│  KVM compute │ Ceph/NFS storage │ OVS/DPDK networking          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Tier-1 telco data center hardware │ 10-25GbE │ SR-IOV NICs    │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Hyperscale telco automation — Tier-1 mobile operators (AT&T, China Mobile, Vodafone) managing end-to-end network service lifecycle with closed-loop automation.

## Pros
- Most complete telco automation platform
- Closed-loop automation (detect → analyze → act automatically)
- Full service lifecycle from design to decommission
- LF Networking project with Tier-1 telco backing

## Cons
- Astronomically complex (40+ microservices)
- Requires massive Kubernetes cluster just to run ONAP itself
- Very long time to production (~18+ months for first deployment)
- "ONAP is a product for building products" — still much manual work

---

# DEPLOYMENT 74 — Bare Metal → Wind River Titanium → Telco VMs/CNFs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TELCO WORKLOADS (Carrier-Grade)                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VNFs (VMs): vEPC │ IMS │ vBNG │ vPGW │ vSGW            │   │
│  │  CNFs (Pods): 5G core │ MEC apps │ ORAN workloads        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              WIND RIVER STUDIO / TITANIUM PLATFORM              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Wind River Studio (Cloud-native lifecycle platform)     │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Based on StarlingX (WR contributes to StarlingX)  │  │   │
│  │  │  Commercial support + certification for telco use  │  │   │
│  │  │  Kubernetes (CNCF certified, WR-tuned)             │  │   │
│  │  │  OpenStack (optional for VM VNFs)                  │  │   │
│  │  │  Wind River proprietary add-ons:                   │  │   │
│  │  │    → Studio Dev (CI/CD pipeline mgmt)              │  │   │
│  │  │    → Studio Ops (fleet management dashboard)       │  │   │
│  │  │    → Lifecycle management tools                    │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              CARRIER-GRADE PERFORMANCE LAYER                    │
│  DPDK │ SR-IOV │ HugePages │ CPU isolation │ PREEMPT_RT         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Telco-grade COTS)             │
│  Intel Xeon │ ECC RAM │ SR-IOV NIC │ 25/100GbE │ BMC           │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Commercial StarlingX for Tier-1 telcos needing SLA guarantees, commercial support, and certification for network functions.

## Pros
- Commercial support with SLA
- Carrier-grade certifications
- StarlingX-based (upstream contribution)
- Wind River ecosystem tools

## Cons
- Expensive proprietary licensing
- Vendor lock-in to Wind River
- StarlingX is available free without Wind River

---

# DEPLOYMENT 75 — Bare Metal → Kubernetes → KubeEdge → Edge Nodes

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              CLOUD MANAGEMENT LAYER                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kubectl (manages both cloud and edge nodes uniformly)   │   │
│  │  Cloud K8s cluster (control plane for all edge nodes)    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  WebSocket tunnel (HTTPS)
┌────────────────────────────▼────────────────────────────────────┐
│              KUBEEDGE CLOUD SIDE (runs in cloud K8s)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CloudCore (KubeEdge cloud component — K8s pod)          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  EdgeHub — manages WebSocket connections to edges  │  │   │
│  │  │  EdgeController — syncs K8s resources to edges     │  │   │
│  │  │  DeviceController — IoT device twin management     │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  WebSocket (works through NAT/firewall)
┌────────────────────────────▼────────────────────────────────────┐
│              EDGE NODE (remote location — may be offline)       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  EdgeCore (KubeEdge edge component)                      │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Edged — local kubelet-like pod manager            │  │   │
│  │  │  EdgeHub — WebSocket client to CloudCore           │  │   │
│  │  │  EventBus — MQTT broker for IoT device events      │  │   │
│  │  │  DeviceTwin — local digital twin for IoT devices   │  │   │
│  │  │  MetaManager — local metadata cache (works offline)│  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Edge Pods (containers on edge node)               │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  AI/ML inference (TensorFlow Lite pod)       │  │  │   │
│  │  │  │  IoT data aggregator pod                     │  │  │   │
│  │  │  │  Protocol adapter pod (Modbus/OPC-UA bridge) │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  IoT DEVICES (connected to edge node)              │  │   │
│  │  │  Sensors │ Cameras │ PLCs │ Industrial equipment   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Edge Hardware)                │
│  Raspberry Pi / Jetson Nano / Industrial PC / ARM SBC           │
│  CPU (ARM64 or x86) │ RAM 2GB+ │ Flash/eMMC │ WiFi/LTE/Eth     │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- CloudCore runs in the central cloud K8s cluster and manages all edge nodes
- EdgeCore runs on each edge device and registers with CloudCore via WebSocket
- EdgeCore's MetaManager caches pod specs and device configs locally — pods continue running even if cloud is unreachable
- DeviceTwin provides a digital twin for each IoT device (reported state + desired state)
- EventBus bridges MQTT from IoT devices into K8s events visible at the cloud level
- kubectl from the cloud can list and manage pods on remote edge nodes

## Use Case
Industrial IoT, smart manufacturing, autonomous vehicles, smart cities — where Kubernetes-managed containers need to run at remote locations with unreliable connectivity.

## Pros
- Works offline — edge pods survive cloud disconnection
- Kubernetes API consistency from cloud to edge
- IoT device twin management built-in
- CNCF project with active community

## Cons
- WebSocket tunnel — limited throughput vs direct K8s
- Limited edge node resources vs cloud K8s features
- Complex debug across cloud-edge boundary
- MQTT bridging adds complexity for IoT

---

# DEPLOYMENT 76 — Bare Metal → KVM → VM (Host) → KVM → Nested VMs (Guest)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              L2 NESTED VM WORKLOADS                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  L2 Nested VM A         L2 Nested VM B                  │   │
│  │  (Ubuntu 22.04)         (Windows 11 — test)             │   │
│  │  vCPU, RAM, disk        vCPU, RAM, disk                 │   │
│  │  (inside L1 VM)         (inside L1 VM)                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  L2 hypervisor (inside L1 VM)
┌────────────────────────────▼────────────────────────────────────┐
│              L1 VIRTUAL MACHINE (Guest OS + KVM)                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Guest OS: Ubuntu 22.04 / RHEL 9                         │   │
│  │  KVM enabled inside guest (nested KVM)                   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  L1 KVM module (kvm.ko inside guest)               │  │   │
│  │  │  QEMU processes for L2 VMs                         │  │   │
│  │  │  libvirt (optional, for L2 VM management)          │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Exposes /dev/kvm to L2 QEMU processes                   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  (L1 VM has nested=on CPU feature exposed by L0 host)          │
└────────────────────────────┬────────────────────────────────────┘
                             │  L0 → L1 VM (shadow EPT tables)
┌────────────────────────────▼────────────────────────────────────┐
│              L0 HOST HYPERVISOR (bare metal)                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KVM + QEMU (L0 level)                                   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  L1 VM configured with:                            │  │   │
│  │  │  cpu: host-passthrough (or -cpu kvm64,vmx=on)      │  │   │
│  │  │  → exposes VMX/SVM instructions to guest           │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Shadow EPT: L0 maps L2 guest memory directly           │   │
│  │  (KVM handles nested EPT page walk — performance cost)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              L0 LINUX KERNEL + KVM MODULE (host)                │
│  kvm.ko │ kvm-intel.ko (nested=1 module param)                  │
│  OR kvm-amd.ko (nested=1)                                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU: Intel VT-x + EPT (nested EPT) or AMD-V + RVI             │
│  RAM: Large pool (L0 + L1 + L2 overhead)                        │
└─────────────────────────────────────────────────────────────────┘

PERFORMANCE OVERHEAD:
┌─────────────────────────────────────────────────────────────────┐
│  L0 (bare metal KVM):    ~2-5% overhead vs native              │
│  L1 (VM in KVM):         ~10-15% overhead vs L0                │
│  L2 (nested VM):         ~40-70% overhead vs L0 (total)        │
│  Memory: 3 EPT page tables maintained simultaneously            │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- L0 (bare metal) KVM host enables nested virtualization: `echo 1 > /sys/module/kvm_intel/parameters/nested`
- L1 VM is created with `cpu=host-passthrough` or `vmx=on` — VMX instructions exposed to guest
- Inside L1, the guest Linux kernel loads kvm.ko and sees /dev/kvm
- L2 VMs are created inside L1 using QEMU
- L0 KVM intercepts L1's VMX instructions and emulates them using shadow EPT tables

## Use Case
Cloud providers offering "bare metal-like" VM instances with VT-x passthrough, CI/CD for hypervisor testing (testing KVM inside a CI VM), and security research.

## Pros
- Test hypervisor environments without physical hardware
- Cloud providers can expose KVM to tenants
- Useful for Kubernetes-in-VM testing
- Vagrant and cloud CI/CD uses this

## Cons
- Significant performance penalty (40-70% at L2)
- Complex memory management (shadow EPT)
- Higher VM exit rate at L2
- Not suitable for production workloads

---

# DEPLOYMENT 77 — Bare Metal → OpenStack → VMs → OpenStack (Nested)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              L2 TENANT VMs (nested cloud)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Tenant L2 VMs (launched via inner OpenStack)            │   │
│  │  These are VMs running inside VMs inside OpenStack       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              INNER OPENSTACK (L1 — running inside L0 VMs)       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Inner OpenStack control plane (Nova, Neutron, Keystone) │   │
│  │  Runs inside L0 Nova VMs (nested KVM enabled)            │   │
│  │  Inner Nova → libvirt → nested KVM → L2 VMs             │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OUTER OPENSTACK L0 (bare metal cloud)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  L0 Nova (provisions L1 OpenStack infra VMs)             │   │
│  │  L0 Neutron (networking for L1 VMs)                      │   │
│  │  L0 VMs: inner-controller, inner-compute × N             │   │
│  │  All with nested KVM enabled (cpu_mode=host-passthrough) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              L0 KVM COMPUTE NODES + LINUX KERNEL                │
│  kvm-intel.ko nested=1 │ libvirt cpu_mode=host-passthrough       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (large — nested overhead 40-70%) │ RAM (massive pool)     │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
OpenStack CI testing (testing OpenStack upgrades using nested OpenStack), development/lab environments, and cloud providers offering "private cloud in a cloud."

## Pros
- Test OpenStack without physical hardware
- Isolated OpenStack environments per project/team
- Useful for telco labs

## Cons
- Extreme overhead (three layers deep)
- Very resource intensive
- Slow I/O and networking at L2
- Production use extremely rare

---

# DEPLOYMENT 78 — Bare Metal → K8s → KubeVirt VMs → Nested K8s

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              INNER KUBERNETES (L2 — inside VMs)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Inner K8s cluster (tenant-controlled)                   │   │
│  │  Pods, services, ingress → all inside VM-based nodes     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  each inner K8s node = a KubeVirt VM
┌────────────────────────────▼────────────────────────────────────┐
│              KUBEVIRT VMs (inner K8s nodes)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VM: inner-ctrl-1   VM: inner-wkr-1   VM: inner-wkr-2   │   │
│  │  Ubuntu 22.04        Ubuntu 22.04      Ubuntu 22.04      │   │
│  │  kubeadm-joined     kubeadm-joined    kubeadm-joined     │   │
│  │  containerd         containerd        containerd         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  KubeVirt manages VMs in outer K8s
┌────────────────────────────▼────────────────────────────────────┐
│              OUTER KUBERNETES (L1 — management cluster)         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KubeVirt operator (manages VMs)                         │   │
│  │  Cluster API (CAPI) + CAPK (KubeVirt provider)           │   │
│  │  → CAPI provisions inner K8s clusters via KubeVirt VMs   │   │
│  │  CDI (disk image import for inner cluster node images)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (outer cluster nodes)           │
│  kvm.ko │ nested KVM (if inner VMs need nested virt)           │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x) │ RAM 512GB+ │ NVMe (Ceph/Longhorn) │ 25GbE NIC   │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Kubernetes-as-a-Service on Kubernetes — hyperscalers offering tenant K8s clusters. CAPK (Cluster API Provider KubeVirt) is the key enabler.

## Pros
- K8s-native multi-cluster provisioning
- Strong VM isolation between tenant clusters
- CAPI provides declarative cluster lifecycle
- Active development by KubeVirt community

## Cons
- VM overhead for K8s nodes
- Complex CAPI + KubeVirt expertise required
- Not yet mainstream production

---

# DEPLOYMENT 79 — Bare Metal → K8s → KubeVirt → VMs (Multi-Layer Combo)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              COMPLEX WORKLOAD LAYER                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container pods (cloud-native apps)                      │   │
│  │  KubeVirt VMs (legacy apps)                              │   │
│  │  Kata container pods (secure multi-tenant containers)    │   │
│  │  → all managed by kubectl simultaneously                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (unified control plane)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RuntimeClass: runc          → regular containers        │   │
│  │  RuntimeClass: kata-qemu     → Kata MicroVM containers   │   │
│  │  VirtualMachine CRD          → KubeVirt full VMs         │   │
│  │                                                          │   │
│  │  All workload types:                                     │   │
│  │  → same kubectl API                                      │   │
│  │  → same networking (CNI / Multus)                        │   │
│  │  → same storage (CSI / PVC)                              │   │
│  │  → same RBAC and security policies                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Storage: Rook-Ceph CSI (RBD for VMs, CephFS for pods)  │   │
│  │  Network: Multus + OVN-K8s (VLANs for VMs)              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + containerd + LINUX KERNEL                    │
│  kvm.ko │ containerd (runc + kata shims) │ QEMU per KubeVirt VM │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x) │ RAM 512GB+ │ NVMe + HDDs │ 25GbE NIC             │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Enterprise consolidation: running containers, secure containers, and full VMs from one Kubernetes control plane. Migration path from VMware while onboarding new cloud-native apps simultaneously.

---

# DEPLOYMENT 80 — Bare Metal → LXD → VMs (QEMU) → Kubernetes (Cluster API)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS (tenant clusters)             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  K8s cluster A (prod)        K8s cluster B (staging)    │   │
│  │  Running inside LXD VMs      Running inside LXD VMs     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s nodes = LXD VMs
┌────────────────────────────▼────────────────────────────────────┐
│              LXD VIRTUAL MACHINES (QEMU-backed)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vm: k8s-ctrl-a-1    vm: k8s-wkr-a-1   vm: k8s-ctrl-b-1 │   │
│  │  Ubuntu 22.04         Ubuntu 22.04      Ubuntu 22.04     │   │
│  │  kubeadm / k3s        kubeadm / k3s     kubeadm / k3s   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  LXD CAPL (Cluster API Provider LXD)
┌────────────────────────────▼────────────────────────────────────┐
│              LXD CLUSTER (management cluster)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  LXD daemon (lxd) on each node                           │   │
│  │  dqlite (Raft consensus) — cluster state                 │   │
│  │  REST API (:8443)                                        │   │
│  │  Storage: ZFS (local) or Ceph (distributed)              │   │
│  │  Network: OVN (SDN for VM networking)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + KVM (each LXD node)                 │
│  kvm.ko │ QEMU (for LXD VMs) │ ZFS │ OVN │ cgroups             │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  LXD Node 1 │ LXD Node 2 │ LXD Node 3 (min for HA)            │
│  CPU (VT-x) │ RAM 256GB+ │ NVMe (ZFS pool) │ 25GbE NIC        │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
LXD-centric infrastructure teams wanting Kubernetes clusters managed via Cluster API, using LXD VMs as K8s nodes.

---

# DEPLOYMENT 81 — Bare Metal → Kubernetes → Neutron-Only + K8s

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Pods with Neutron-provided networking                   │   │
│  │  (Kuryr CNI: K8s pods on Neutron subnets)               │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Kuryr CNI driver
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (CNI = Kuryr → Neutron)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kuryr-controller (K8s → Neutron translation)            │   │
│  │  kuryr-cni (pod NIC provisioning via Neutron ports)      │   │
│  │  → K8s Services → Neutron LBaaS (Octavia)               │   │
│  │  → K8s NetworkPolicy → Neutron Security Groups          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK NEUTRON (network only)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  neutron-server │ ML2 plugin │ OVN backend               │   │
│  │  Provides: virtual networks, routers, FIPs, SGs, LBaaS  │   │
│  │  WITHOUT running full OpenStack (no Nova, no Cinder)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + OVN                                 │
│  OVS (Open vSwitch) │ OVN (Open Virtual Network)               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  K8s nodes │ Neutron network nodes │ 10GbE NIC                 │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Using OpenStack Neutron's rich networking (FIPs, security groups, LBaaS) for Kubernetes pod networking without running full OpenStack. Useful when migrating from OpenStack to K8s while retaining network infrastructure.

---

# DEPLOYMENT 82 — Bare Metal → seL4 → Formally Verified Partitions

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              PARTITIONED WORKLOADS                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Partition A (Safety Critical)    Partition B (General)  │   │
│  │  ┌──────────────────────────┐     ┌──────────────────┐   │   │
│  │  │  Certified RTOS / app    │     │  Linux / apps    │   │   │
│  │  │  Medical device control  │     │  (UI, comms)     │   │   │
│  │  │  Flight control system   │     │                  │   │   │
│  │  └──────────────────────────┘     └──────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              seL4 MICROKERNEL HYPERVISOR                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  THE FORMALLY VERIFIED MICROKERNEL                       │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Kernel = ~10,000 lines of C (tiny — proven safe)  │  │   │
│  │  │                                                     │  │   │
│  │  │  Formal proofs (Isabelle/HOL theorem prover):       │  │   │
│  │  │  ✓ Functional correctness (does what spec says)    │  │   │
│  │  │  ✓ No buffer overflows                             │  │   │
│  │  │  ✓ No NULL pointer dereferences                   │  │   │
│  │  │  ✓ No arithmetic overflows                         │  │   │
│  │  │  ✓ Stack bounded                                   │  │   │
│  │  │                                                     │  │   │
│  │  │  Capability-based access control                   │  │   │
│  │  │  (objects accessed only via unforgeable caps)       │  │   │
│  │  │                                                     │  │   │
│  │  │  CAmkES / Microkit (component framework on seL4)   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  VM Extensions (ARM SMMUv3 / x86 IOMMU for device iso)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              HARDWARE (ARM / RISC-V / x86)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ARM Cortex-A / RISC-V SoC (preferred — smaller TCB)    │   │
│  │  IOMMU (required for device isolation guarantees)        │   │
│  │  RAM │ Flash │ Peripherals                               │   │
│  │              BARE METAL                                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

FORMAL VERIFICATION PIPELINE:
┌─────────────────────────────────────────────────────────────────┐
│  seL4 C source → binary translation proof → Isabelle/HOL proofs │
│  → machine-checked proofs (billions of proof steps verified)    │
│  TSYS (verified) → Security Properties: confidentiality,       │
│  integrity, and availability proofs per partition               │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- seL4 is the world's only formally verified OS kernel — every line proven correct in Isabelle/HOL
- Capability-based access control prevents any partition from accessing another's memory without an explicit capability
- IOMMU extends isolation to DMA-capable devices — even peripherals can't violate partition boundaries
- CAmkES/Microkit provides component frameworks for building systems on seL4
- The formal proofs verify: no buffer overflow, no undefined behavior, stack bounded — in the kernel

## Use Case
Military systems, medical devices (pacemakers, insulin pumps), aviation, and any domain where a security/safety compromise is catastrophically unacceptable.

## Pros
- Only formally verified OS kernel in production
- Mathematical proof of security properties
- Minimal attack surface (~10K lines)
- Used in DARPA HACMS, Boeing aircraft, autonomous vehicle systems

## Cons
- Extremely specialized development (CAmkES, proof-aware)
- Very small developer pool
- Performance limited by microkernel IPC overhead
- Most commercial software cannot run directly

---

# DEPLOYMENT 83 — Bare Metal → Jailhouse → Linux + RTOS Partitions

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              PARTITIONED WORKLOADS                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Cell 0 (Root Cell)          Cell 1 (Inmate)             │   │
│  │  ┌──────────────────────┐    ┌──────────────────────┐    │   │
│  │  │  Linux (full OS)     │    │  RTOS (bare metal C)  │    │   │
│  │  │  Runs normal Linux   │    │  Hard RT task control  │    │   │
│  │  │  workloads           │    │  Industrial PLC logic  │    │   │
│  │  │  Manages Jailhouse   │    │  Motor controller      │    │   │
│  │  │  (jailhouse CLI)     │    │  No OS overhead        │    │   │
│  │  └──────────────────────┘    └──────────────────────┘    │   │
│  │  Partitioned CPU cores:                                  │   │
│  │  Cell 0: cores 0-5  │  Cell 1: cores 6-7 (dedicated RT) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              JAILHOUSE HYPERVISOR                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Jailhouse (loaded as Linux kernel module)               │   │
│  │  Type-1 static partitioning hypervisor                  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Static memory partitioning (no dynamic allocation) │  │   │
│  │  │  CPU core assignment (per-cell dedicated cores)     │  │   │
│  │  │  Device ownership (per-cell device assignment)      │  │   │
│  │  │  No scheduling — cells run simultaneously           │  │   │
│  │  │  Comm regions (shared memory between cells)         │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Very small hypervisor: ~12,000 lines of C code          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  loads over running Linux kernel
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (pre-Jailhouse load)                  │
│  Linux boots normally → jailhouse enable → Jailhouse takes over │
│  Linux demoted to Cell 0 root cell                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  x86 multi-core / ARM multi-core (tested on i.MX8, Jetson)     │
│  Intel VT-x / VT-d required for x86                            │
│  RAM │ Peripherals divided between cells                        │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Linux boots normally, then `jailhouse enable cell0.cfg` loads the Jailhouse hypervisor
- Jailhouse takes over hardware and demotes Linux to the "root cell"
- Inmate cells are created: `jailhouse cell create rtos-cell.cfg`
- Cells get dedicated CPU cores, memory regions, and devices — no sharing
- RTOS inmate runs on dedicated cores with zero interference from Linux

## Use Case
Industrial automation, robotics, and automotive — running Linux (for general compute + networking) and a hard RT RTOS (for motor control, sensor reading) on the same SoC.

## Pros
- Hard real-time in inmate cell with zero Linux interference
- Small hypervisor code (auditable)
- Existing Linux system can be "Jailhoused" without full redesign
- Good Siemens/industry backing

## Cons
- Static partitioning — cannot change partition config at runtime
- Requires hardware VT-x/VT-d
- Limited to shared-memory IPC between cells
- Complex board support configuration files

---

# DEPLOYMENT 84 — Bare Metal → Barrelfish → Multi-Kernel Research OS

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              RESEARCH APPLICATIONS                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Research workloads (custom Barrelfish apps)             │   │
│  │  Network stack experiments │ OS research │ HPC protos    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              BARRELFISH (Multi-Kernel OS)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  One CPU Core = One CPU Driver (like a separate kernel)  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Core 0: CPU driver 0 (manages core 0 resources)   │  │   │
│  │  │  Core 1: CPU driver 1 (manages core 1 resources)   │  │   │
│  │  │  Core 2: CPU driver 2 ...                          │  │   │
│  │  │  Core N: CPU driver N                              │  │   │
│  │  │  → Each core runs its own kernel-like driver       │  │   │
│  │  │  → No shared memory between cores (message passing)│  │   │
│  │  │  → System Knowledge Base (SKB) — distributed fact  │  │   │
│  │  │    database for system configuration               │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Inter-core communication: URPC (userspace RPC, lockless)│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Many-core x86/ARM (designed for 1000+ cores)                   │
│  NUMA topology │ Large RAM │ NIC                               │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Academic OS research, particularly for heterogeneous many-core systems. ETH Zurich + Microsoft Research project.

## Pros / Cons
- Research-only — not for production
- Groundbreaking architecture ideas (multi-kernel model)

---

# DEPLOYMENT 85 — Bare Metal → NOVA Microhypervisor → VMs (Research)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              GUEST VMs                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Linux guest │ Other OS guests                           │   │
│  │  Runs via NOVA's thin hypervisor abstraction             │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              NOVA MICROHYPERVISOR                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  NOVA: Minimalist Type-1 microhypervisor                 │   │
│  │  Trusted Computing Base (TCB) < 10,000 lines             │   │
│  │  Capability-based access control (like seL4)             │   │
│  │  Runs on Intel VT-x hardware                             │   │
│  │  User-level VMMs (each VM has a user-space VMM process)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (x86)                          │
│  Intel VT-x required │ RAM │ NIC                               │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Security research, trusted computing, and base for Genode OS framework.

---

# DEPLOYMENT 86 — Bare Metal → Xen Research Forks → Experimental VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              EXPERIMENTAL VM WORKLOADS                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  HVM / PV / PVH guests (various virtualization modes)    │   │
│  │  Research: unikernel VMs │ hardened VMs │ micro-VMs      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              XEN RESEARCH FORK / EXTENDED XEN                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Examples:                                               │   │
│  │  XenServer (Citrix commercialization of Xen)             │   │
│  │  Xen on ARM (for mobile/embedded research)               │   │
│  │  XAPI-based toolstack variants                           │   │
│  │  PVH (PV + HVM hybrid) mode research                     │   │
│  │  Dom0-less Xen (no privileged Dom0 — distributed control)│   │
│  │                                                          │   │
│  │  Dom0-less: multiple driver domains, no single Dom0      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              XEN HYPERVISOR (below all OSes)                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  x86 (primary) / ARM (growing) │ RAM │ NIC                    │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Academic research on hypervisor security, embedded Xen (Xilinx Zynq UltraScale+), and Dom0-less designs for car/industrial systems.

---

# DEPLOYMENT 87 — Bare Metal → Talos Linux → K8s (Pure Immutable OS)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Application pods │ System pods (CNI, CSI, monitoring)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              TALOS LINUX (immutable OS)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  machined (PID 1, gRPC API server — replaces init/systemd│   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  No shell (no bash/sh)                             │  │   │
│  │  │  No SSH daemon                                     │  │   │
│  │  │  No package manager (apt/yum)                      │  │   │
│  │  │  No interactive login                              │  │   │
│  │  │  Read-only /usr filesystem (squashfs)              │  │   │
│  │  │  All config via talosctl (gRPC, mTLS)              │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  KUBERNETES (bootstrapped via Talos)               │  │   │
│  │  │  containerd → static pods → K8s control plane      │  │   │
│  │  │  kubelet → worker node agent                       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  OS UPDATE MODEL (A/B partition schema)            │  │   │
│  │  │  Partition A (current) │ Partition B (new version) │  │   │
│  │  │  talosctl upgrade → writes B, reboots into B       │  │   │
│  │  │  Rollback: if unhealthy → boot back to A           │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              TALOS LINUX KERNEL (custom minimal build)          │
│  Minimal config │ CIS hardened │ No debug symbols │ FIPS mode   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Any x86_64 / ARM64 hardware │ UEFI preferred                  │
│  PXE boot supported │ Cloud images (AWS/GCP/Azure/metal)        │
└─────────────────────────────────────────────────────────────────┘

TALOSCTL WORKFLOW (no SSH required):
┌─────────────────────────────────────────────────────────────────┐
│  Developer laptop → talosctl apply-config → node gRPC API       │
│  talosctl upgrade → A/B OS partition upgrade                    │
│  talosctl kubeconfig → get K8s credentials                      │
│  talosctl dmesg / talosctl logs → view logs (no SSH needed)     │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Talos boots from an immutable squashfs root — nothing can modify /usr
- machined (PID 1) replaces systemd/init and serves a gRPC API over mTLS
- All management is done via talosctl — no SSH, no shell, no interactive access
- K8s is bootstrapped from machined's control — kubelet and static pods start automatically
- OS upgrades: talosctl writes new OS image to inactive A/B partition and reboots

## Use Case
Security-critical Kubernetes environments needing zero shell attack surface. Compliance-heavy industries (fintech, healthcare), air-gapped deployments, and GitOps-everything teams.

## Pros
- Zero shell access — minimal attack surface
- CIS-hardened kernel by default
- A/B atomic upgrades — no manual patching
- FIPS 140-2 mode available
- GitOps-native

## Cons
- No traditional Linux admin tools (very different mental model)
- Debugging requires container-based ephemeral pods
- Smaller ecosystem than standard K8s distros
- Talos-specific toolchain learning curve

---

# DEPLOYMENT 88 — Bare Metal → Flatcar Container Linux → K8s

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container pods │ System pods                            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              FLATCAR CONTAINER LINUX                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  (CoreOS Container Linux successor — maintained by SUSE) │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Immutable read-only /usr (verified dm-verity)     │  │   │
│  │  │  A/B partition update system (Nebraska/Omaha)      │  │   │
│  │  │  systemd (PID 1) — limited to services             │  │   │
│  │  │  SSH access available (unlike Talos)               │  │   │
│  │  │  No package manager (static OS)                    │  │   │
│  │  │  Container-first: Docker, containerd built-in      │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Ignition / Butane (provisioning config at first   │  │   │
│  │  │  boot — NIC config, hostname, k8s join token, etc.)│  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Kubernetes (kubeadm / k3s / RKE2 installed atop Flatcar)│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (Flatcar hardened)                    │
│  systemd │ containerd │ dm-verity (root FS verification)        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  x86_64 / ARM64 │ BIOS/UEFI │ PXE boot or disk image          │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Immutable-OS Kubernetes nodes for organizations that want CoreOS-style stability, SUSE backing, and SSH access (unlike Talos). Popular for managed K8s node pools.

## Pros
- Immutable OS — zero drift possible
- A/B atomic updates (auto-update or controlled)
- SUSE commercial support
- SSH accessible (easier than Talos for ops teams)
- Used by Kinvolk (SUSE acquisition), Microsoft AKS nodes

## Cons
- Less restrictive than Talos (SSH exists)
- Smaller ecosystem than Ubuntu/RHEL for node OS
- Some software expects traditional package manager

---

# DEPLOYMENT 89 — Bare Metal → NixOS → Declarative K8s Nodes

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Pods │ services │ all K8s workloads                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              NixOS (declarative OS)                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  configuration.nix or flake.nix (declares EVERYTHING)   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  services.kubernetes.enable = true;                │  │   │
│  │  │  services.kubernetes.roles = ["master" "node"];    │  │   │
│  │  │  networking.firewall.allowedPorts = [6443 ...];    │  │   │
│  │  │                                                    │  │   │
│  │  │  Nix package manager: 80,000+ reproducible pkgs   │  │   │
│  │  │  nixos-rebuild switch → atomic OS reconfig        │  │   │
│  │  │  nixos-rebuild rollback → revert any change       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  REPRODUCIBILITY:                                  │  │   │
│  │  │  Same configuration.nix → identical system state  │  │   │
│  │  │  anywhere (like Dockerfile but for entire OS)     │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (NixOS-managed)                       │
│  Nix store: /nix/store (immutable, content-addressed packages)  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  x86_64 / ARM64 │ Any hardware with Linux driver support       │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Research teams and DevOps practitioners wanting fully reproducible, declarative OS + K8s node configuration as code. Gaining traction in tech companies with strong infrastructure-as-code culture.

## Pros
- Fully declarative — config file = complete system description
- Atomic upgrades and rollbacks
- Reproducible builds (same config = identical system)
- 80,000+ packages in nixpkgs

## Cons
- Steep learning curve (Nix language is unusual)
- Small community vs Ubuntu/RHEL
- Some proprietary software hard to package
- Not common in enterprise

---

# DEPLOYMENT 90 — Bare Metal → Nutanix AHV + Prism → HCI Extended

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              ENTERPRISE WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VMs: ERP (SAP), databases (Oracle), VDI (Citrix)        │   │
│  │  Containers: Nutanix Kubernetes Engine (NKE clusters)    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              NUTANIX MANAGEMENT PLANE                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Prism Central (central multi-cluster management)        │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Prism Pro: Advanced analytics + capacity planning  │  │   │
│  │  │  Calm: Infrastructure-as-Code (Blueprints)          │  │   │
│  │  │  Flow: Micro-segmentation (L4-L7 network policy)    │  │   │
│  │  │  Leap: DR / Business Continuity                     │  │   │
│  │  │  NKE: Nutanix Kubernetes Engine (K8s clusters)      │  │   │
│  │  │  Era: Database-as-a-Service (Oracle/MSSQL/PG)       │  │   │
│  │  │  X-Play: IT process automation                      │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Prism Element (per-cluster management)                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              NUTANIX NODE (AHV + CVM per node)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  AHV HYPERVISOR (KVM-based)                              │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Guest VMs (Windows, Linux, SAP, Oracle)           │  │   │
│  │  │  NKE K8s cluster nodes (VMs)                       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  CVM (Controller VM — storage)                           │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Stargate (I/O path)    Curator (cluster ops)      │  │   │
│  │  │  Cassandra (metadata)   Zookeeper (coordination)   │  │   │
│  │  │  Hades (disk ops)       Arithmos (stats)           │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  distributed NVM storage fabric
┌────────────────────────────▼────────────────────────────────────┐
│              NUTANIX DISTRIBUTED STORAGE (AOS)                  │
│  NVMe SSD tier + HDD capacity tier │ Replication Factor 2/3    │
│  Data locality (CVM reads local disk first)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Nutanix NX / Dell XC / HPE DX / Lenovo HX certified platforms  │
│  CPU: Intel Xeon / AMD EPYC │ RAM: ECC │ NVMe+HDD │ 10/25GbE  │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Full enterprise HCI for organizations consolidating compute + storage + networking on certified hardware with full Nutanix support. VMware replacement leader.

## Pros
- Complete enterprise HCI with full stack: VMs + K8s + DBaaS + DR + automation
- Hardware-agnostic (run on multiple OEM platforms)
- Excellent support + SLA
- NKE for K8s alongside VMs
- Era for database lifecycle automation

## Cons
- Very expensive per-node licensing
- Nutanix software and ecosystem lock-in
- AHV historically less feature-rich than VMware ESXi (gap closing)
- Prism Central requires dedicated resources

---

# DEPLOYMENT 91 — Bare Metal → VMware vSphere + Tanzu → Kubernetes + VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS (VMware unified platform)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Traditional VMs (Windows, Linux — vSphere managed)      │   │
│  │  Kubernetes clusters (Tanzu managed K8s)                 │   │
│  │  Containers (Tanzu Application Platform — Cloud Foundry) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              VMWARE TANZU + vSPHERE MANAGEMENT                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vCenter Server (VM + Tanzu cluster management)          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Tanzu Kubernetes Grid (TKG)                       │  │   │
│  │  │  → provisions K8s clusters on vSphere VMs          │  │   │
│  │  │  vSphere with Tanzu (Supervisor)                   │  │   │
│  │  │  → K8s API directly on vSphere (no separate TKG)  │  │   │
│  │  │  Tanzu Application Platform (TAP)                  │  │   │
│  │  │  → developer self-service, supply chain security   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  vSphere features: DRS │ HA │ vMotion │ vSAN │ NSX-T     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              ESXI HOSTS (hypervisor nodes)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Guest VMs (Windows/Linux workloads)                     │   │
│  │  Tanzu K8s cluster node VMs (Ubuntu-based)               │   │
│  │  VMkernel (ESXi hypervisor kernel)                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vSAN: local NVMe pooled across ESXi hosts               │   │
│  │  NSX-T: software-defined networking (micro-segmentation) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  VMware HCL-certified hardware                                  │
│  Intel Xeon / AMD EPYC │ RAM 512GB+ │ NVMe (vSAN) │ 25GbE NIC │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Large enterprises standardized on VMware wanting to modernize to Kubernetes while retaining existing VMware investment. Bridges traditional VM world to cloud-native.

## Pros
- Unified VMware management for VMs + K8s
- Mature HA, DRS, vMotion for all workloads
- NSX-T micro-segmentation for both VMs and K8s
- vSAN for integrated storage

## Cons
- Dramatically increased cost post-Broadcom 2024 acquisition
- Very complex licensing tiers
- Driving customers away toward Nutanix, Proxmox, OpenShift
- Deep VMware lock-in

---

# DEPLOYMENT 92 — Bare Metal → Azure Stack HCI → Windows/Linux VMs + AKS

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Windows Server VMs │ Linux VMs │ AKS Arc K8s clusters   │   │
│  │  Azure Arc-enabled data services (Arc SQL, Arc PG)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              AZURE STACK HCI MANAGEMENT                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Azure Portal (manages on-premises HCI via Azure Arc)    │   │
│  │  Windows Admin Center (local web UI)                     │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  AKS on Azure Stack HCI (K8s clusters)             │  │   │
│  │  │  Azure Arc (extends Azure mgmt to on-premises)      │  │   │
│  │  │  Azure Monitor (telemetry to Azure cloud)           │  │   │
│  │  │  Microsoft Defender for Cloud (security posture)    │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              AZURE STACK HCI OS (on each node)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Hyper-V (VMs) │ S2D (Storage Spaces Direct)             │   │
│  │  SDN (Software Defined Networking — via Windows SDN)     │   │
│  │  Cluster Services (Windows Server Failover Clustering)   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  Storage Spaces Direct (S2D) — pools local NVMe across nodes    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Azure Stack HCI certified hardware (600+ solutions listed)     │
│  Intel Xeon / AMD EPYC │ RAM ECC │ NVMe (S2D) │ 10/25GbE RoCE │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Microsoft-centric enterprises running on-premises HCI with Azure cloud management — hybrid cloud strategy bridging on-premises with Azure services.

## Pros
- Azure portal management for on-premises workloads
- Azure Arc extends cloud services on-premises
- Native Windows + Hyper-V support
- Microsoft support SLA

## Cons
- Microsoft Azure subscription required (ongoing cost)
- Windows-centric — Linux is secondary
- Certified hardware list restricts choices
- Complex licensing (Azure Stack HCI + Windows Server + Arc)

---

# DEPLOYMENT 93 — Bare Metal → Proxmox + Ceph → HCI (Full Stack)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS (unified HCI)                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KVM VMs (Windows, Linux, legacy apps)                   │   │
│  │  LXC containers (lightweight Linux services)             │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              PROXMOX VE (web UI port 8006)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ┌──────────────────┐  ┌────────────────────────────┐    │   │
│  │  │ VM Management    │  │  Cluster & HA Management   │    │   │
│  │  │ (KVM + LXC)      │  │  (corosync + pve-ha-mgr)   │    │   │
│  │  └──────────────────┘  └────────────────────────────┘    │   │
│  │  ┌──────────────────────────────────────────────────────┐ │   │
│  │  │  STORAGE: BUILT-IN CEPH (pveceph)                    │ │   │
│  │  │  Proxmox manages entire Ceph lifecycle:              │ │   │
│  │  │  pveceph init → pveceph createmon → pveceph createosd│ │   │
│  │  │  Ceph Dashboard embedded in Proxmox web UI          │ │   │
│  │  │  ┌────────────────────────────────────────────────┐  │ │   │
│  │  │  │  Ceph MONs │ MGRs │ OSDs │ MDS │ RGW           │  │ │   │
│  │  │  │  (all running on same Proxmox nodes)            │  │ │   │
│  │  │  └────────────────────────────────────────────────┘  │ │   │
│  │  │  RBD storage pool → VM disks (live migratable)       │ │   │
│  │  │  CephFS storage → ISO images, backups, shared        │ │   │
│  │  └──────────────────────────────────────────────────────┘ │   │
│  │  ┌──────────────────────────────────────────────────────┐ │   │
│  │  │  NETWORK: Linux bridge / VLAN-aware / OVS            │ │   │
│  │  └──────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  (Proxmox + Ceph run on same hardware)
┌────────────────────────────▼────────────────────────────────────┐
│              PROXMOX CLUSTER (3+ nodes for Ceph quorum)         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1 (Proxmox + Ceph MON/OSD)                        │   │
│  │  Node 2 (Proxmox + Ceph MON/OSD)                        │   │
│  │  Node 3 (Proxmox + Ceph MON/OSD)                        │   │
│  │  Corosync (cluster comms) ←→ dqlite (Raft, pve cluster) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL + CEPH (each node)              │
│  kvm.ko │ ceph-osd (per disk) │ ceph-mon │ ceph-mgr             │
│  LVM (local) │ ZFS (local, optional)                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Node 1              Node 2              Node 3 (minimum 3)    │
│  CPU 16c+            CPU 16c+            CPU 16c+              │
│  RAM 128GB+ ECC      RAM 128GB+ ECC      RAM 128GB+ ECC        │
│  NVMe: OS+KVM        NVMe: OS+KVM        NVMe: OS+KVM          │
│  HDDs: Ceph OSDs     HDDs: Ceph OSDs     HDDs: Ceph OSDs       │
│  NIC: 2×10GbE        NIC: 2×10GbE        NIC: 2×10GbE         │
│  (1 for VM traffic, 1 for Ceph replication)                    │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Proxmox integrates Ceph deployment directly via `pveceph` commands in the web UI
- All nodes run both Proxmox hypervisor (KVM/LXC) and Ceph storage daemons (MON/OSD/MGR)
- VM disks are stored as Ceph RBD volumes — enabling live migration without shared SAN
- Proxmox HA manager uses corosync for cluster communication and pve-ha-mgr for VM failover
- When a node fails: corosync detects it → HA manager restarts VMs on surviving nodes (from Ceph RBD)

## Use Case
Most popular open-source VMware alternative for SMBs and cost-conscious enterprises. 3-node minimum for full HA + Ceph — total cost of hardware only.

## Pros
- Completely free (no licensing cost)
- Integrated Ceph via web UI — no Ceph expertise needed for basics
- Live migration with Ceph RBD storage
- HA VM failover
- PBS (Proxmox Backup Server) integration

## Cons
- Ceph at scale still requires expertise
- Proxmox web UI limited for large (100+ node) clusters
- No native K8s integration (use Harvester or KubeVirt separately)
- 3-node minimum adds cost

---

# DEPLOYMENT 94 — Bare Metal → Red Hat Virtualization (EOL) Migration Path

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS (being migrated)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VMs currently on RHV → migrating to:                   │   │
│  │  Option A: OpenShift Virtualization (KubeVirt)           │   │
│  │  Option B: Proxmox VE                                    │   │
│  │  Option C: oVirt (community RHV fork)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              RHV (LEGACY — EOL June 2024)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RHV Manager (Java EE — oVirt Engine)                    │   │
│  │  RHVH hosts (RHEL-based hypervisor)                      │   │
│  │  KVM/QEMU (hypervisor backend)                           │   │
│  │  GlusterFS or NFS (shared storage)                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│  STATUS: End of Life as of June 2024                            │
│  Replacement: OpenShift Virtualization (enterprise)             │
│  OR: oVirt (community support, no Red Hat SLA)                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  RHEL-certified hardware                                        │
└─────────────────────────────────────────────────────────────────┘

MIGRATION DECISION TREE:
┌─────────────────────────────────────────────────────────────────┐
│  RHV Customer                                                   │
│  ├── Red Hat customer, want full support → OpenShift Virt ($$)  │
│  ├── Budget conscious, Linux VM focused  → oVirt (community)   │
│  ├── Want simplest migration, no vendor → Proxmox VE (free)    │
│  └── Already K8s-first               → KubeVirt on vanilla K8s │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
RHV customers need to migrate — this documents the migration path analysis rather than a live deployment.

---

# DEPLOYMENT 95 — Bare Metal → Ceph + OpenStack + K8s + Metal3 (Hyperscaler)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MULTI-TIER WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Tenant VMs (OpenStack tenants via Horizon/Nova)         │   │
│  │  Bare Metal nodes (OpenStack Ironic tenants)             │   │
│  │  Kubernetes clusters (Magnum / external K8s on Ironic)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LAYER 3: OPENSTACK (IaaS control plane)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Nova (KVM VMs) │ Ironic (bare metal) │ Neutron (SDN)    │   │
│  │  Cinder (→ Ceph RBD) │ Glance (→ Ceph RBD) │ Swift (RGW) │   │
│  │  Magnum (K8s-as-a-Service) │ Heat (orchestration)        │   │
│  │  Keystone │ Barbican │ Designate │ Octavia               │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LAYER 2: MANAGEMENT KUBERNETES CLUSTER             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenStack control plane pods (OpenStack-Helm / OSO)     │   │
│  │  Metal3 + Ironic (bare metal provisioning)               │   │
│  │  Cluster API (manages K8s cluster provisioning)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LAYER 1: CEPH STORAGE CLUSTER                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RBD (block) → Cinder volumes, Nova ephemeral, Glance    │   │
│  │  RGW (object) → Swift API, backup storage                │   │
│  │  CephFS (file) → Manila shared filesystems               │   │
│  │  50+ OSD nodes, petabytes of storage                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LAYER 0: BARE METAL INFRASTRUCTURE                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Control nodes:  CPU 32c │ RAM 256GB │ NVMe │ 25GbE      │   │
│  │  Compute nodes:  CPU 128c│ RAM 1TB+  │ NVMe │ 25GbE×2   │   │
│  │  Storage nodes:  CPU 16c │ RAM 64GB  │ 12×HDD+2×NVMe    │   │
│  │                          │           │ 25GbE×2           │   │
│  │  Network nodes:  CPU 8c  │ RAM 32GB  │ 100GbE spine      │   │
│  │  Managed via: Metal3 / MAAS / Ironic standalone          │   │
│  └──────────────────────────────────────────────────────────┘   │
│  Physical network:                                              │
│  Leaf-spine topology │ BGP EVPN │ 100GbE spine │ 25GbE leaf   │
└─────────────────────────────────────────────────────────────────┘

FULL CALL CHAIN:
kubectl apply BareMetalHost → Metal3/Ironic powers on node → PXE
→ OS deploy (RHEL/Ubuntu) → Ironic provisions → Nova registers
→ Tenant requests VM → Nova schedules → KVM VM → Ceph RBD disk
```

## How It Works
- This is the extreme full-stack: Kubernetes manages OpenStack which manages KVM VMs and bare metal, all backed by Ceph
- Metal3 + Ironic handle all physical node provisioning via Redfish/IPMI
- OpenStack control plane pods run in the management K8s cluster (self-healing via K8s)
- All storage (VM disks, images, objects, shared filesystems) routes through Ceph
- Tenant interface: OpenStack Horizon or Nova/Cinder API — they get VMs, bare metal, K8s clusters

## Use Case
Large private cloud operators (national research networks, telcos, universities with 1000+ servers) building AWS/Azure-scale private infrastructure.

## Pros
- Complete private cloud IaaS with every service
- Bare metal, VM, and K8s workloads from one API
- No vendor lock-in (all open source)
- Scales to thousands of nodes

## Cons
- Requires multiple dedicated teams (Ceph, OpenStack, K8s, Networking, HW)
- Years to design, deploy, and stabilize
- Operational complexity is extreme
- Not for organizations under 500 servers

---

# DEPLOYMENT 96 — Bare Metal → K8s → OpenStack Control Plane (OCP-on-K8s)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT VMs (OpenStack instances)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Tenant VMs provisioned by Nova API                      │   │
│  │  Running on bare metal compute nodes with KVM            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  nova-compute → libvirt → KVM on BM
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK SERVICES (as K8s pods)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Deployed via: OpenStack-Helm OR OpenStack Operator      │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  keystone deploy   │ keystone pod(s) in K8s      │    │   │
│  │  │  nova deploy       │ nova-api, nova-scheduler,   │    │   │
│  │  │                    │ nova-conductor pods          │    │   │
│  │  │  neutron deploy    │ neutron-server pods          │    │   │
│  │  │  cinder deploy     │ cinder-api, cinder-vol pods  │    │   │
│  │  │  glance deploy     │ glance-api pods              │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Infrastructure pods (StatefulSets):             │    │   │
│  │  │  MariaDB-Galera-0,1,2 │ RabbitMQ-0,1,2          │    │   │
│  │  │  Memcached-0,1,2      │ Ingress controller       │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s manages OpenStack pods
┌────────────────────────────▼────────────────────────────────────┐
│              MANAGEMENT KUBERNETES CLUSTER                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kube-apiserver │ etcd │ scheduler │ controllers          │   │
│  │  Ceph CSI (persistent storage for MariaDB, RabbitMQ)     │   │
│  │  Multus CNI (provider network access for Neutron)        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              BARE METAL COMPUTE NODES (outside K8s)             │
│  nova-compute │ neutron-agent │ libvirt │ KVM                   │
│  (join OpenStack but NOT managed by K8s)                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  K8s control nodes: CPU 32c │ RAM 256GB │ NVMe │ 25GbE         │
│  Compute nodes: CPU 128c │ RAM 1TB+ │ NVMe │ 25GbE×2          │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
AT&T, Verizon-style deployments where OpenStack is the tenant API but K8s manages the control plane for GitOps-driven operations and self-healing.

---

# DEPLOYMENT 97 — Bare Metal → OpenNebula + K8s + KubeVirt + Ceph (Full)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenNebula VMs (traditional IaaS tenants)               │   │
│  │  Kubernetes pods (via OneKE — OpenNebula K8s Engine)     │   │
│  │  KubeVirt VMs (inside K8s)                               │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENNEBULA (oned)                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Sunstone (web UI) │ onevm CLI │ Terraform provider      │   │
│  │  OneKE (K8s cluster deployment via ONE templates)        │   │
│  │  Scheduler │ VM lifecycle │ image/network management     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (OneKE-provisioned) + KUBEVIRT          │
│  KubeVirt for VM-in-K8s │ Rook-Ceph for K8s storage           │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              HYPERVISOR NODES (KVM via ONE)                     │
│  KVM + libvirt │ Ceph RBD client (VM disks) │ OpenNebula agent │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              CEPH STORAGE CLUSTER                               │
│  RBD (VM + K8s disks) │ RGW (object) │ CephFS (shared)        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Compute + storage nodes │ CPU │ RAM │ HDDs/NVMe │ 10GbE       │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
European research clouds (EGI, GÉANT ecosystem) wanting an OpenStack-lighter alternative with full VM + K8s + storage capabilities.

---

# DEPLOYMENT 98 — Bare Metal → XCP-ng + Ceph + CloudStack

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT WORKLOADS                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Tenant VMs (Windows/Linux) via CloudStack self-service  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              CLOUDSTACK (management + multi-tenancy)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CloudStack Management Server (Java)                     │   │
│  │  Zone → Pod → Cluster (XCP-ng) → Hosts                  │   │
│  │  Primary Storage: Ceph RBD (via CloudStack Ceph plugin)  │   │
│  │  Secondary Storage: Ceph RGW (S3-compat for ISO/templates│   │
│  │  CloudStack agent on each XCP-ng Dom0                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              XCP-ng HOSTS (Xen hypervisor)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Dom0 (control) + DomU guest VMs                         │   │
│  │  XAPI toolstack │ CloudStack agent                       │   │
│  │  Ceph RBD client (VM disks from Ceph)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              CEPH CLUSTER                                       │
│  MON │ OSD │ MGR │ RGW (S3 for secondary storage)              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  XCP-ng compute nodes │ Ceph storage nodes │ Controller        │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Asia-Pacific cloud hosting providers using CloudStack's AWS API compatibility with XCP-ng (free Xen) and Ceph for a cost-effective private cloud stack.

---

# DEPLOYMENT 99 — Bare Metal → Nomad + Firecracker → MicroVM Workloads

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MIXED WORKLOADS (Nomad-scheduled)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container jobs (docker driver)                          │   │
│  │  Firecracker MicroVM jobs (firecracker-task-driver)      │   │
│  │  Binary jobs (exec driver)                               │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  job placement
┌────────────────────────────▼────────────────────────────────────┐
│              NOMAD (orchestration)                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Nomad Servers (Raft consensus)                          │   │
│  │  Nomad Clients (worker nodes)                            │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Task Drivers:                                     │  │   │
│  │  │  docker → containerd (standard containers)         │  │   │
│  │  │  firecracker-task-driver → Firecracker MicroVMs    │  │   │
│  │  │  exec → raw binaries (host process)                │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Consul (service discovery + health checks)              │   │
│  │  Vault (secrets injection into tasks)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              FIRECRACKER (per MicroVM task)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  jailer + firecracker binary per MicroVM task            │   │
│  │  virtio-net (TAP) │ virtio-blk │ minimal Linux kernel    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/kvm
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL                                 │
│  kvm.ko │ TUN/TAP │ cgroups │ seccomp (jailer)                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V) │ RAM │ NVMe │ NIC                          │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
HashiCorp-ecosystem teams wanting MicroVM isolation without Kubernetes complexity — Nomad's simplicity + Firecracker's security for mixed workload platforms.

## Pros
- Nomad's simplicity vs K8s complexity
- Firecracker's strong isolation for security-sensitive tasks
- Docker + MicroVM + binary workloads from one scheduler
- Consul + Vault integration

## Cons
- Experimental Firecracker driver for Nomad (community, not official)
- Much smaller ecosystem than K8s + Kata/Firecracker
- Limited orchestration features vs K8s

---

# DEPLOYMENT 100 — Bare Metal → StarlingX + Ceph → Edge Distributed Cloud

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              CENTRAL CLOUD                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  StarlingX System Controller (large data center)         │   │
│  │  dcmanager (Distributed Cloud Manager)                   │   │
│  │  Ceph cluster (central storage for bootstrapping)        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  WAN (HTTPS) → subclouds
┌────────────────────────────▼────────────────────────────────────┐
│              REMOTE EDGE SUBCLOUDS (100s of sites)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Subcloud A (telecom central office)                     │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  StarlingX AIO-DX (2 servers)                      │  │   │
│  │  │  K8s (embedded) → CNF workloads                   │  │   │
│  │  │  OpenStack (optional) → VNF workloads             │  │   │
│  │  │  Ceph (2-node, min replication) → local storage   │  │   │
│  │  │  DPDK + SR-IOV → line-rate network functions      │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Subcloud B (remote retail site)                         │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  StarlingX AIO-SX (1 server — no HA)              │  │   │
│  │  │  K8s → edge app pods (POS, inventory, ML inference)│  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  all managed from central cloud
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (each site)                    │
│  AIO-DX: 2× Intel Xeon servers (SR-IOV NIC, 25GbE, NVMe)       │
│  AIO-SX: 1× Edge server (lower spec, 16GB RAM min)              │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Telco distributed cloud — managing hundreds of edge sites (central offices, cell towers, retail) from one central cloud controller. Canadian open source project (Wind River, Intel, VMware contributions).

---

# DEPLOYMENT 101 — IBM Z Mainframe → z/VM → Linux → K8s → Containers

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS (on mainframe)                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes pods (Java, Node.js, Python apps)            │   │
│  │  Mainframe-native workloads (COBOL, z/OS services)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (on Linux on Z)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Red Hat OpenShift on IBM Z (S390X architecture)         │   │
│  │  OR upstream K8s on Ubuntu/RHEL on Z                    │   │
│  │  containerd │ CRI-O │ s390x container images            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s runs inside Linux on z/VM guest
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX ON Z (z/VM guests)                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RHEL for Z (s390x) │ Ubuntu on Z                        │   │
│  │  Multiple Linux guests on same z/VM                      │   │
│  │  K8s control plane VMs + K8s worker VMs                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              z/VM HYPERVISOR                                    │
│  CP (Control Program) │ CMS │ thousands of Linux guests        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              IBM Z HARDWARE (z16 / LinuxONE III)                │
│  S390X CPUs │ Pervasive Encryption │ Up to 32TB RAM            │
│  DASD / Flash Express storage │ Redundant everything           │
│                  BARE METAL (IBM Z)                             │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Banks and insurance companies running Kubernetes workloads directly on mainframe — combining cloud-native container applications with COBOL batch processing on the same hardware.

## Pros
- K8s on mainframe hardware — 6-nines availability
- Pervasive encryption for all container data
- Same hardware as core banking systems
- Massive consolidation — thousands of Linux VMs + K8s on one machine

## Cons
- IBM Z licensing and hardware costs are extreme
- S390X architecture limits container image availability
- Very specialized expertise

---

# DEPLOYMENT 102 — Bare Metal → ACRN + StarlingX → Industrial Edge (Experimental)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MIXED-CRITICALITY EDGE WORKLOADS                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RTOS partition (hard RT — motor control, PLC)           │   │
│  │  Linux/K8s partition (general compute — ML, monitoring)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              STARLINGX (K8s/OpenStack management)               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Manages the general compute partition (Linux side)      │   │
│  │  K8s for cloud-native edge workloads                     │   │
│  │  Coordinates with ACRN for partition lifecycle           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              ACRN HYPERVISOR (Intel IoT/auto type-1)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Service OS (SOS) — Linux, manages UOS lifecycle         │   │
│  │  User OS (UOS) — RT partition for industrial control     │   │
│  │  LAPIC passthrough — native timer interrupts for RT      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Intel VT-x (ACRN requires Intel)
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Intel Edge SoC)               │
│  Intel Atom / Core / Xeon-D │ VT-x │ RAM │ Industrial I/O      │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Intel-backed experimental architecture for autonomous vehicle or factory automation — hard RT workloads + cloud-native management on same edge hardware. **Research/experimental status.**

---

# DEPLOYMENT 103 — Bare Metal → K8s → OpenFaaS + Firecracker → Serverless

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              FUNCTION INVOCATIONS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  REST → Gateway → Function (Python/Go/Node.js)           │   │
│  │  Each invocation: Firecracker MicroVM start → exec → end │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + OpenFaaS + Kata-Firecracker           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  faas-netes (OpenFaaS K8s backend)                       │   │
│  │  Functions deployed as RuntimeClass: kata-firecracker    │   │
│  │  Auto-scale to zero / scale up based on Prometheus       │   │
│  │  NATS queue for async invocations                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KATA RUNTIME → FIRECRACKER VMM                     │
│  kata-runtime shim → firecracker binary → MicroVM per pod      │
│  jailer (cgroup + seccomp sandbox around Firecracker)           │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V) │ RAM (density: 1000s of MicroVMs) │ NVMe    │
└─────────────────────────────────────────────────────────────────┘
```

---

# DEPLOYMENT 104 — Bare Metal → Xen + KubeVirt + Ceph (Unified)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KubeVirt VMs (managed by K8s API, backed by Ceph RBD)   │   │
│  │  K8s container pods (standard OCI containers)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + KUBEVIRT (inside Xen Dom0)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Xen Dom0 running Linux + K8s control plane              │   │
│  │  KubeVirt operator managing VM lifecycle                 │   │
│  │  KubeVirt VMs run as QEMU processes inside Dom0          │   │
│  │  (or as Xen DomU guests via KubeVirt Xen driver)        │   │
│  │  Rook-Ceph CSI for VM disk storage (PVCs)                │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              XEN HYPERVISOR + CEPH                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Xen provides L1 isolation below Dom0                    │   │
│  │  Ceph cluster (replicated block storage for all VMs)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V for Xen HVM) │ RAM │ OSD disks │ NIC         │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Organizations already on Xen (XCP-ng) wanting to adopt Kubernetes-native VM management via KubeVirt while retaining Xen's isolation properties.

---

# DEPLOYMENT 105 — Bare Metal → Kubernetes → Multi-CNI + SR-IOV → NFV Pods

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              NFV / CNF WORKLOADS                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vRouter CNF Pod (requires 3 network interfaces):        │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  eth0: management net (default K8s CNI - Calico)   │  │   │
│  │  │  net1: SR-IOV VF (data plane - line rate DPDK)     │  │   │
│  │  │  net2: SR-IOV VF (data plane - second interface)   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES WITH MULTUS CNI                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Multus CNI (meta-plugin — enables multiple NICs per pod)│   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  NetworkAttachmentDefinition CRD                   │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  Primary: Calico CNI (management traffic)     │  │  │   │
│  │  │  │  Secondary: SR-IOV device plugin              │  │  │   │
│  │  │  │  (allocates VFs from physical NIC to pods)   │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  SR-IOV Device Plugin (DaemonSet — manages VF allocation) │   │
│  │  CPU Manager (pin vRouter pod CPUs to specific cores)    │   │
│  │  HugePages (for DPDK memory in vRouter pod)              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              HOST OS PERFORMANCE LAYER                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  SR-IOV NIC: 16+ VFs per physical port                  │   │
│  │  DPDK (inside pod, user-space network I/O)               │   │
│  │  HugePages: 1GB pages for DPDK memory pool               │   │
│  │  CPU isolation: cores reserved for DPDK poll mode driver │   │
│  │  IOMMU (VT-d): required for SR-IOV VF passthrough        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + IOMMU                               │
│  VFIO (Virtual Function I/O) driver │ SR-IOV kernel support    │
│  intel_iommu=on │ isolcpus= kernel params                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU: Intel Xeon (DPDK-optimized, NUMA-aware)                  │
│  RAM: 256GB+ with HugePages configured                         │
│  NIC: Intel X710 / XXV710 / E810 (SR-IOV, 25GbE+)             │
│  BIOS: VT-x + VT-d enabled │ HugePages │ CPU frequency fixed   │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Multus CNI enables pods to have multiple network interfaces (K8s default CNI + additional)
- NetworkAttachmentDefinition (CRD) defines each additional network's CNI plugin config
- SR-IOV Device Plugin discovers NIC VFs and advertises them as Kubernetes resources (`intel.com/intel_sriov_netdevice`)
- Pods request SR-IOV VFs via resource requests — Multus attaches them at pod creation
- Inside the pod, DPDK bypasses the kernel and polls the VF directly for line-rate performance

## Use Case
5G user plane functions (UPF), packet gateways, vRouter CNFs requiring line-rate packet processing (40Gbps+) while still managed by Kubernetes.

## Pros
- Kubernetes API for carrier-grade network functions
- SR-IOV for near-native NIC performance inside pods
- DPDK inside containers achieves line-rate
- Multus supports arbitrary number of interfaces per pod

## Cons
- BIOS and kernel tuning required on every node
- SR-IOV limits number of pods per NIC (typically 32-128 VFs)
- DPDK in pods requires CPU core pinning — reduces scheduling flexibility
- Complex NetworkAttachmentDefinition configuration

---

# DEPLOYMENT 106 — Bare Metal → LXD + Kubernetes → Developer Cloud

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              DEVELOPER WORKLOADS                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Developer K8s namespaces (isolated per team)            │   │
│  │  CI/CD pipelines (Jenkins / GitLab / Tekton)             │   │
│  │  Dev/test environments (ephemeral, fast-delete)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (dev cluster)                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kubeadm or k3s deployed inside LXD containers or VMs   │   │
│  │  Namespace isolation per team                            │   │
│  │  RBAC → team-scoped access                              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s nodes = LXD containers/VMs
┌────────────────────────────▼────────────────────────────────────┐
│              LXD (rapid instance provisioning)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  LXC containers (K8s worker nodes — fast spin-up)        │   │
│  │  LXD VMs (K8s control plane — stronger isolation)        │   │
│  │  ZFS storage (instant clones for new dev environments)   │   │
│  │  OVN networking (per-team VLAN isolation)                │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + KVM (for LXD VMs)                   │
│  ZFS │ KVM │ OVN │ cgroups v2 │ namespaces                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x for VMs) │ RAM │ NVMe (ZFS pool) │ 10GbE NIC        │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Developer platforms, internal K8s as a service, CI/CD infrastructure where speed of new environment creation is critical.

## Pros
- LXD container spin-up in seconds (vs minutes for full VM)
- ZFS instant clone → new K8s node in seconds
- Low overhead per developer namespace
- OVN provides proper network isolation

## Cons
- LXD container K8s nodes share kernel (less isolation than VMs)
- LXD VMs have more overhead
- Not suitable for production multi-tenant

---

# DEPLOYMENT 107 — Bare Metal → Full GitOps Stack (Talos + Flux + Crossplane)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              EVERYTHING-AS-CODE (Git is source of truth)        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Git repositories (GitLab / GitHub)                      │   │
│  │  infra-repo/      app-repo/       platform-repo/         │   │
│  │  (Talos config)   (K8s manifests) (Crossplane XRDs)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  GitOps reconciliation (push/pull)
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + GITOPS LAYER                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  FluxCD (GitOps operator)                                │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  source-controller → watch Git repo               │  │   │
│  │  │  kustomize-controller → apply Kustomize manifests │  │   │
│  │  │  helm-controller → manage Helm chart releases      │  │   │
│  │  │  notification-controller → alerts + webhooks       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Crossplane (Infrastructure as Code inside K8s)          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  XRD (Composite Resource Definitions)              │  │   │
│  │  │  → provision cloud resources as K8s CRDs           │  │   │
│  │  │  → databases, DNS, S3 buckets via kubectl apply    │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  APPLICATION WORKLOADS (reconciled by Flux)             │   │
│  │  Pods │ Services │ Ingress │ PVCs                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  nodes managed by Talos API
┌────────────────────────────▼────────────────────────────────────┐
│              TALOS LINUX NODES (immutable OS)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  machined (API server) │ kubelet │ containerd             │   │
│  │  talosctl apply-config (from infra-repo in Git)          │   │
│  │  A/B OS partition upgrade (from infra-repo pipeline)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (Talos) + HARDWARE                    │
└────────────────────────────┬────────────────────────────────────┘
                             │  Metal3/iPXE for initial provisioning
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  PXE/iPXE boot → Talos installer from Git-triggered pipeline   │
│  CPU │ RAM │ NVMe │ NIC │ BMC (Redfish for Metal3 control)     │
└─────────────────────────────────────────────────────────────────┘

WORKFLOW: Git commit → CI pipeline → Flux reconciles → cluster
          changes applied → no human touch after initial boot
```

## How It Works
- Talos config (machineconfig YAML) lives in infra-repo → any node config change is a Git PR
- Flux watches app-repo and applies K8s manifests automatically on every commit
- Crossplane provisions external infrastructure (DNS, DBs) via K8s CRDs — all in Git
- Metal3 provisions new bare metal nodes from Git-defined BareMetalHost resources
- Zero manual node access: no SSH, no kubectl on dev laptops in production

## Use Case
Cloud-native-first teams and SRE platforms wanting zero-touch infrastructure with full auditability and reproducibility. Growing adoption in fintech and security-sensitive environments.

## Pros
- 100% Git-driven — every change is auditable
- No snowflake servers — any node can be rebuilt from Git
- Crossplane enables cloud + bare metal from same Git repo
- Flux handles app deployment + infrastructure

## Cons
- Steep learning curve (Talos + Flux + Crossplane simultaneously)
- Bootstrap chicken-and-egg problem (need running K8s to run Flux)
- Crossplane providers still maturing
- Debugging complex reconciliation chains

---

# DEPLOYMENT 108 — Bare Metal → Kubernetes + KubeVirt → VMware Migration

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS (VMware VMs migrated to KubeVirt)        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VMs originally from VMware (Windows/Linux)              │   │
│  │  Now running as KubeVirt VMIs in Kubernetes              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + KUBEVIRT                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MTV (Migration Toolkit for Virtualization)               │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  MTV Operator (Red Hat / community)                │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  Provider: vSphere → K8s+KubeVirt            │  │  │   │
│  │  │  │  VDDK (VMware Data Disk Kit) → import VMDK   │  │  │   │
│  │  │  │  CDI imports disk → Ceph RBD PVC             │  │  │   │
│  │  │  │  VirtualMachine object created               │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  KubeVirt VMs (post-migration)                           │   │
│  │  virtctl console / virtctl ssh → access migrated VMs    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              SOURCE: VMWARE vSPHERE (during migration)          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ESXi hosts running existing VMs                         │   │
│  │  VMDK disk images (being read by MTV)                    │   │
│  │  vCenter API (source of VM inventory for MTV)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (target)                       │
│  New Kubernetes worker nodes                                    │
│  CPU (VT-x/AMD-V) │ RAM │ Ceph (VM disk storage) │ 25GbE NIC  │
└─────────────────────────────────────────────────────────────────┘

MIGRATION FLOW:
vCenter inventory → MTV discovers VMs → MTV copies VMDK (via VDDK)
→ CDI imports to Ceph RBD PVC → KubeVirt VMI created → VM boots
→ Validate → cutover (source VM powered off) → migration complete
```

## How It Works
- MTV (Migration Toolkit for Virtualization) connects to vCenter as a "Provider"
- VDDK allows MTV to read VMware VMDK disk images
- CDI (Containerized Data Importer) imports the disk image into a Ceph RBD PVC
- KubeVirt creates a VirtualMachine object pointing to the imported PVC
- VM boots in KubeVirt — testing phase; then cutover (power off source, run target permanently)

## Use Case
Organizations migrating off VMware after Broadcom's 2024 price increases. Red Hat OpenShift Virtualization is the main commercial implementation.

## Pros
- Automated VMware-to-KubeVirt migration
- Minimal manual intervention
- Existing VMware admins can use familiar VM concepts in KubeVirt
- Open source (MTV is community + Red Hat)

## Cons
- VDDK has complex licensing
- Large VM disks = long migration time
- Network compatibility between VMware and KubeVirt networking requires planning
- Windows VMs need driver changes post-migration

---

# DEPLOYMENT 109 — Full Stack: Bare Metal → MAAS → Juju → Charmed OpenStack + K8s

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              CLOUD WORKLOADS                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Tenant VMs (OpenStack Nova KVM VMs)                     │   │
│  │  Charmed Kubernetes clusters (Juju-deployed K8s)         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              CHARMED OPENSTACK (Juju-deployed)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Juju Charms (software operators for each service)       │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  nova-cloud-controller charm                       │  │   │
│  │  │  nova-compute charm (on each compute node)         │  │   │
│  │  │  neutron-api charm (Neutron controller)            │  │   │
│  │  │  keystone charm │ cinder charm │ glance charm       │  │   │
│  │  │  ceph-mon charm │ ceph-osd charm                   │  │   │
│  │  │  percona-cluster charm (MariaDB Galera)            │  │   │
│  │  │  rabbitmq-server charm                             │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Juju controller manages charm lifecycle + relations     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Juju deploys charms via SSH to MAAS nodes
┌────────────────────────────▼────────────────────────────────────┐
│              JUJU (service lifecycle manager)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Juju Controller (manages all Juju operations)           │   │
│  │  juju deploy charmed-openstack-bundle                    │   │
│  │  juju add-unit nova-compute -n 5 (add 5 compute nodes)  │   │
│  │  juju upgrade-charm nova-compute (rolling upgrade)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Juju provisions via MAAS
┌────────────────────────────▼────────────────────────────────────┐
│              MAAS (Metal-as-a-Service)                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MAAS Region Controller (API + DB)                       │   │
│  │  MAAS Rack Controller (DHCP + TFTP per rack)             │   │
│  │  → Commissions nodes (discovers hardware)                │   │
│  │  → Deploys Ubuntu on nodes requested by Juju             │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  IPMI/Redfish power control + PXE
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL NODE POOL                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  All hardware managed by MAAS:                           │   │
│  │  Controller nodes (3) │ Compute nodes (N) │ OSD nodes (N)│   │
│  │  Network nodes (2)    │ Storage nodes     │               │   │
│  │  BMC (iDRAC/iLO/IPMI/Redfish) on all nodes               │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

DEPLOYMENT FLOW:
1. MAAS discovers hardware (PXE commission)
2. Juju adds MAAS as cloud
3. juju deploy charmed-openstack-bundle
4. Juju allocates MAAS nodes, deploys Ubuntu, runs charms
5. Charms configure OpenStack services and relations between them
6. OpenStack ready in hours (vs days manually)
```

## How It Works
- MAAS discovers and commissions all bare metal hardware
- Juju uses MAAS as its cloud provider — requests machines, deploys Ubuntu, runs charm agents
- Charmed OpenStack bundle deploys ~20 charms that configure and relate all OpenStack services
- Juju relations wire services together (e.g., nova-compute → rabbitmq-server charm relation)
- Upgrades: `juju upgrade-charm` performs rolling upgrades with zero downtime

## Use Case
Canonical's reference architecture for OpenStack. Banks, telecoms, and research networks wanting a vendor-supported, automated OpenStack deployment on bare metal.

## Pros
- Fully automated deployment (hours, not weeks)
- Juju handles rolling upgrades automatically
- MAAS handles hardware lifecycle
- Canonical commercial support
- Full Ubuntu/Canonical stack

## Cons
- Ubuntu/Canonical ecosystem only
- Juju learning curve
- Charm quality varies by community charm
- Less common outside Canonical ecosystem

---

# DEPLOYMENT 110 — Bare Metal → Kubernetes → OpenFaaS → Firecracker → Serverless (Full Pipeline)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              DEVELOPER / CLIENT LAYER                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Function developer (faas-cli deploy --gateway https://..│   │
│  │  End user client (HTTP POST /function/myFunction)         │   │
│  │  CI/CD pipeline (auto-deploy on git push)                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENFAAS PLATFORM LAYER                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenFaaS Gateway (K8s Deployment)                       │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Reverse proxy → routes to function replicas       │  │   │
│  │  │  Scale-to-zero: scales function pods to 0 when idle│  │   │
│  │  │  Scale-from-zero: starts pods on first request     │  │   │
│  │  │  Authentication (basic/JWT)                        │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Queue-worker (NATS Streaming)                           │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Async invocations → NATS topic → worker queues    │  │   │
│  │  │  Retry logic │ Dead letter queue                    │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Prometheus (metrics) → KEDA (event-driven autoscaling) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  function pods with RuntimeClass
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES SCHEDULING + RUNTIME LAYER              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Function A Pod         Function B Pod       Function C  │   │
│  │  RuntimeClass:          RuntimeClass:         RuntimeCls: │   │
│  │  kata-firecracker       kata-qemu (fallback)  runc       │   │
│  │  ┌──────────────────┐   ┌────────────────┐  ┌─────────┐ │   │
│  │  │Firecracker MicroVM│  │QEMU MicroVM    │  │Container│ │   │
│  │  │Own Linux kernel   │  │(device compat) │  │(shared  │ │   │
│  │  │~125ms cold start  │  │~300ms cold     │  │ kernel) │ │   │
│  │  │<5MB VMM overhead  │  │~30MB overhead  │  │<1MB     │ │   │
│  │  └──────────────────┘   └────────────────┘  └─────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Kata runtime creates VMs
┌────────────────────────────▼────────────────────────────────────┐
│              KATA CONTAINERS RUNTIME LAYER                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kata-runtime (containerd shim v2)                       │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  kata-fc backend → launches Firecracker VMM        │  │   │
│  │  │  kata-qemu backend → launches QEMU (fallback)      │  │   │
│  │  │  Kata agent (inside each VM) → OCI compliance      │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              FIRECRACKER + JAILER LAYER                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Per-function MicroVM:                                   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  jailer process (cgroup + seccomp + chroot sandbox) │  │   │
│  │  │  └── firecracker VMM (Rust, ~1MB binary)           │  │   │
│  │  │       └── MicroVM:                                  │  │   │
│  │  │            Minimal Linux kernel (~5.10 LTS)         │  │   │
│  │  │            Shared read-only rootfs (overlay)        │  │   │
│  │  │            Per-VM writable layer (overlayfs)        │  │   │
│  │  │            virtio-net (TAP device)                  │  │   │
│  │  │            virtio-blk (function code disk)          │  │   │
│  │  │            Kata agent (receives container cmds)     │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/kvm ioctls
┌────────────────────────────▼────────────────────────────────────┐
│              KVM KERNEL MODULE                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kvm.ko + kvm-intel.ko / kvm-amd.ko                      │   │
│  │  EPT (Extended Page Tables) — hardware memory isolation  │   │
│  │  VMX/SVM — hardware CPU virtualization                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (HOST OS)                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  TUN/TAP devices (one per MicroVM for virtio-net)        │   │
│  │  cgroups v2 (resource limits per jailer/MicroVM)         │   │
│  │  seccomp (syscall filter for Firecracker process)        │   │
│  │  overlayfs (shared rootfs + per-VM writable layer)       │   │
│  │  iptables/eBPF (pod network rules via CNI plugin)        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              STORAGE + NETWORK LAYER                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Function code: container registry → init container      │   │
│  │                 → mounts as virtio-blk in MicroVM        │   │
│  │  Shared rootfs:  cached on each node (NVMe)              │   │
│  │  Pod networking: Calico/Cilium CNI → eBPF routing        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Host Servers)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CPU: Intel VT-x or AMD-V (mandatory for KVM)           │   │
│  │  Cores: 64+ (many concurrent MicroVMs need CPU headroom)│   │
│  │  RAM: 256GB+ (1000 MicroVMs × 128MB avg = 128GB)        │   │
│  │  NVMe: Fast local (rootfs snapshot caching)             │   │
│  │  NIC: 25GbE+ (TAP device I/O for many MicroVMs)         │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

COMPLETE INVOCATION TRACE:
HTTP POST /function/imageResizer
  → OpenFaaS Gateway pod
    → routes to imageResizer pod (or creates new if scale-to-zero)
      → K8s creates pod with RuntimeClass: kata-firecracker
        → containerd calls kata-runtime shim
          → kata-runtime calls Firecracker API (via REST on socket)
            → jailer creates cgroup/seccomp sandbox
              → Firecracker creates MicroVM (boots in ~125ms)
                → Kata agent receives OCI start command
                  → function process starts
                    → image resize executes
                      → response returned up the stack
                        → HTTP 200 to client

COLD START BREAKDOWN:
├── K8s pod scheduling:     ~50ms
├── containerd shim start:  ~10ms
├── Firecracker VMM start:  ~30ms
├── MicroVM boot:           ~125ms
├── Kata agent ready:       ~20ms
└── Function init:          ~50-200ms (language-dependent)
    TOTAL cold start:       ~285-435ms

WARM INVOCATION (existing MicroVM):
└── Function call:          ~1-5ms
```

## How It Works
- OpenFaaS Gateway receives all function HTTP requests and routes them to appropriate pods
- Scale-to-zero: when no traffic, OpenFaaS scales function pods to 0 replicas
- On first request (cold start): K8s schedules new pod with kata-firecracker RuntimeClass
- containerd calls kata-runtime shim which starts a Firecracker VMM
- The jailer sandboxes Firecracker (cgroup + seccomp + chroot)
- Firecracker boots a minimal Linux kernel in ~125ms
- Kata agent inside the VM receives the OCI container start command
- Function process starts inside the VM and processes the request
- Response travels back through the same chain

## Use Case
Self-hosted serverless platform for organizations requiring strong security isolation per function invocation (multi-tenant SaaS, security-sensitive compute), data residency requirements, or cost optimization at high volume vs public cloud.

## Pros
- KVM hardware isolation per function invocation
- ~125ms cold start (competitive with public cloud FaaS)
- Full control over hardware, networking, and data
- Open source entire stack (Apache 2.0)
- Scale-to-zero reduces cost at low load
- NATS async queuing for high-throughput workloads
- Per-function resource limits via cgroups

## Cons
- Extremely complex multi-layer stack to debug and operate
- Cold start slower than pure container FaaS (~285-435ms vs ~50ms)
- KVM hardware support required on all nodes
- Kata + Firecracker expertise rare
- KEDA + Prometheus tuning needed for good autoscaling behavior
- Not as managed/turnkey as AWS Lambda

---

# FINAL QUICK REFERENCE TABLE: Deployments 71–110

| # | Stack | Box Layers (Bottom→Top) | Best For | Complexity |
|---|-------|------------------------|---------|-----------|
| 71 | StarlingX | Bare Metal → DPDK OS → StarlingX → K8s+OpenStack | Telco 5G edge | Extreme |
| 72 | K8s → OSM MANO | Bare Metal → K8s + OpenStack → OSM → CNFs/VNFs | ETSI NFV orchestration | Very High |
| 73 | OpenStack + ONAP | Bare Metal → OpenStack+K8s → ONAP → VNFs | Tier-1 telco automation | Extreme |
| 74 | Wind River Titanium | Bare Metal → WR OS → WR Studio → K8s+OpenStack | Commercial StarlingX | Extreme |
| 75 | K8s → KubeEdge | Bare Metal (edge) → Linux → KubeEdge → IoT pods | Industrial IoT edge | High |
| 76 | KVM nested | Bare Metal → KVM L0 → VM L1 → KVM L1 → VM L2 | Dev/CI nested virt | High |
| 77 | OpenStack nested | Bare Metal → OpenStack L0 → VMs → OpenStack L1 | Testing, lab clouds | Extreme |
| 78 | K8s → KubeVirt → K8s | Bare Metal → K8s → KubeVirt VMs → inner K8s | K8s-as-a-Service | Very High |
| 79 | K8s multi-runtime | Bare Metal → K8s → runc+kata+KubeVirt simultaneously | Enterprise migration | Very High |
| 80 | LXD → K8s (CAPI) | Bare Metal → LXD VMs → K8s (CAPI-managed) | LXD-centric platform | High |
| 81 | K8s + Neutron-only | Bare Metal → K8s → Kuryr → Neutron networking | OpenStack-to-K8s net migration | High |
| 82 | seL4 | Bare Metal → seL4 microkernel → certified partitions | Military/medical security | Extreme |
| 83 | Jailhouse | Bare Metal → Linux → Jailhouse → RT + Linux cells | Industrial automation | High |
| 84 | Barrelfish | Bare Metal → Barrelfish multi-kernel | Research many-core OS | Research |
| 85 | NOVA microhypervisor | Bare Metal → NOVA → VMs | Security research | Research |
| 86 | Xen research forks | Bare Metal → Xen (Dom0-less / ARM / PVH) | Hypervisor research | Research |
| 87 | Talos K8s | Bare Metal → Talos Linux → K8s | Security-first K8s | High |
| 88 | Flatcar K8s | Bare Metal → Flatcar → K8s | Immutable OS K8s nodes | Medium |
| 89 | NixOS K8s | Bare Metal → NixOS → K8s | Declarative OS + K8s | High |
| 90 | Nutanix AHV Extended | Bare Metal → AOS → AHV+CVM → VMs+NKE | Enterprise HCI full stack | High |
| 91 | VMware Tanzu | Bare Metal → ESXi → vSphere+Tanzu → VMs+K8s | VMware enterprise cloud | Very High |
| 92 | Azure Stack HCI | Bare Metal → HCI OS → Hyper-V+S2D → VMs+AKS | Microsoft hybrid cloud | High |
| 93 | Proxmox + Ceph HCI | Bare Metal → Proxmox+Ceph → KVM VMs+LXC | OSS VMware alternative | High |
| 94 | RHV → Migration | Bare Metal → RHV (EOL) → migration path | RHV customers (deprecated) | N/A |
| 95 | Ceph+OS+K8s+Metal3 | Bare Metal → Metal3 → OS+K8s → Ceph | Hyperscale private cloud | Extreme |
| 96 | K8s → OS control plane | Bare Metal → K8s → OpenStack pods → KVM | GitOps OpenStack control | Extreme |
| 97 | OpenNebula+K8s+Ceph | Bare Metal → Ceph → OpenNebula → K8s+VMs | EU research cloud | Very High |
| 98 | XCP-ng+Ceph+CloudStack | Bare Metal → Ceph → XCP-ng → CloudStack | Hosting provider | Very High |
| 99 | Nomad + Firecracker | Bare Metal → Linux → Nomad → docker+Firecracker | HashiCorp + MicroVM | High |
| 100 | StarlingX + Ceph edge | Bare Metal → StarlingX → K8s → CNFs (multi-site) | Telco distributed cloud | Extreme |
| 101 | IBM Z + z/VM + K8s | IBM Z → z/VM → Linux on Z → K8s → Pods | Mainframe cloud-native | Very High |
| 102 | ACRN + StarlingX | Intel Edge → ACRN → StarlingX+K8s → mixed | Industrial edge (experimental) | Extreme |
| 103 | K8s+OpenFaaS+FC | Bare Metal → K8s → OpenFaaS → Kata-FC → FaaS | Self-hosted serverless | Very High |
| 104 | Xen+KubeVirt+Ceph | Bare Metal → Xen → K8s+KubeVirt → Ceph VMs | Xen-to-K8s migration | Very High |
| 105 | K8s+Multus+SRIOV | Bare Metal → Linux+DPDK → K8s → Multus → CNFs | 5G UPF, vRouter CNFs | Very High |
| 106 | LXD+K8s dev cloud | Bare Metal → LXD → K8s → dev namespaces | Developer platform | Medium |
| 107 | Talos+Flux+Crossplane | Bare Metal → Talos → K8s → Flux+Crossplane | 100% GitOps platform | Very High |
| 108 | K8s+KubeVirt VMware migration | Bare Metal → K8s+KubeVirt+MTV → migrated VMs | VMware off-ramp | High |
| 109 | MAAS+Juju+CharmedOS | Bare Metal → MAAS → Juju → Charmed OpenStack | Canonical private cloud | Very High |
| 110 | K8s+OpenFaaS+FC (full) | Bare Metal → KVM → K8s → Kata → Firecracker → FaaS | Full serverless platform | Extreme |

---

# MASTER COMPLEXITY SUMMARY (All 110 Deployments)

```
LOW COMPLEXITY (1-5 steps):
  #1 LXC, #3 systemd-nspawn, #6 NetBSD, #18 QEMU-TCG,
  #28 Scale HC3, #30 k3s

MEDIUM COMPLEXITY (operational but manageable):
  #2 LXD, #4 FreeBSD Jails, #7 KVM+QEMU, #8 libvirt,
  #10 ESXi, #11 Hyper-V, #12 Proxmox, #13 Oracle VM,
  #16 bhyve, #20 PowerVM, #25 Nutanix, #26 HPE,
  #27 FusionCompute, #29 Kubernetes, #31 Docker Swarm,
  #32 Nomad, #33 Apptainer, #37 LXD unified, #45 MicroShift,
  #54 Warewulf, #60 GlusterFS+KVM, #88 Flatcar, #93 Proxmox+Ceph

HIGH COMPLEXITY (requires specialist expertise):
  #5 OpenBSD VMM, #9 Xen, #14 oVirt, #15 XCP-ng, #21 QNX,
  #23 PikeOS, #24 ACRN, #34 KVM→VMs→K8s, #35 KVM→VMs→Docker,
  #36 LXD→K8s nodes, #38 KubeVirt, #39 Kata, #40 Firecracker-K8s,
  #41 Harvester, #43 Talos, #44 k3s+KubeVirt, #47 Ironic,
  #51 MAAS, #52 Metal3, #53 Ironic-standalone, #55 CloudStack,
  #61 LINSTOR, #62 Firecracker-standalone, #63 Cloud-Hypervisor,
  #66 Xen+MirageOS, #67 IncludeOS, #68 OSv, #69 Unikraft,
  #75 KubeEdge, #76 KVM-nested, #80 LXD→K8s-CAPI, #83 Jailhouse,
  #87 Talos, #89 NixOS, #90 Nutanix-Extended, #99 Nomad+FC,
  #105 Multus+SRIOV, #106 LXD+K8s-dev, #108 VMware-migration

VERY HIGH COMPLEXITY (dedicated team required):
  #19 IBM z/VM, #22 INTEGRITY, #42 OpenShift+KubeVirt, #46 OpenStack,
  #48 Magnum, #49 OpenStack→VMs→K8s, #56 OpenNebula, #58 Ceph+OS,
  #59 Ceph+K8s+KubeVirt, #64 OpenFaaS+Firecracker, #72 OSM MANO,
  #74 Wind River, #78 K8s→KubeVirt→K8s, #79 K8s multi-runtime,
  #81 K8s+Neutron, #91 VMware Tanzu, #92 Azure Stack HCI,
  #97 OpenNebula+K8s+Ceph, #98 XCP-ng+Ceph+CloudStack,
  #103 K8s+OpenFaaS+FC, #104 Xen+KubeVirt+Ceph, #107 Talos+Flux+Crossplane,
  #109 MAAS+Juju+CharmedOS

EXTREME COMPLEXITY (months/years to master):
  #50 K8s→OpenStack-pods, #65 Knative+KubeVirt, #70 OPNFV,
  #71 StarlingX, #73 ONAP, #77 OpenStack-nested, #82 seL4,
  #95 Ceph+OS+K8s+Metal3, #96 K8s→OS-control-plane,
  #100 StarlingX+Ceph-edge, #101 IBM-Z+K8s, #102 ACRN+StarlingX,
  #110 Full serverless pipeline

RESEARCH / EXPERIMENTAL:
  #57 Eucalyptus (deprecated), #84 Barrelfish, #85 NOVA-microhypervisor,
  #86 Xen-research-forks
```

---
*Part 3 of 3 — Deployments 71–110 with complete box architectures*
*Combined with Part 1 (1–30) and Part 2 (31–70): all 110 deployments documented*
