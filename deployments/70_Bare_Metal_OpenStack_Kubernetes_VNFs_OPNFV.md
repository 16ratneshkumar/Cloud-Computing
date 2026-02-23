[← Back to Index](../index.md)

# DEPLOYMENT 70 — Bare Metal → OpenStack + Kubernetes → VNFs (OPNFV)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                  VNFs (OPNFV)                  │
├──────────────────────────────────────────────────┤
│             OpenStack + Kubernetes             │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              NETWORK SERVICES (end-user perspective)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Mobile subscribers │ Internet customers │ Enterprise VPN │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              NFV SERVICE LAYER                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MANO (Management and Orchestration)                     │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  NFV Orchestrator (NFVO) — OSM / ONAP              │  │   │
│  │  │  VNF Manager (VNFM) — per-VNF lifecycle            │  │   │
│  │  │  Virtualized Infrastructure Manager (VIM)          │  │   │
│  │  │  → OpenStack (for VM-based VNFs)                   │  │   │
│  │  │  → Kubernetes (for container-based CNFs)            │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  NETWORK FUNCTIONS (VNFs / CNFs)                         │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  VM-based VNFs (on OpenStack):                     │  │   │
│  │  │  vFirewall │ vLoadBalancer │ vRouter │ vIMS        │  │   │
│  │  │  vEPC (mobile core) │ vBNG │ vCPE                  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  CNFs (on Kubernetes):                             │  │   │
│  │  │  free5GC (5G core) │ Open5GS │ SD-Core             │  │   │
│  │  │  Contiv-VPP (fast data path)                       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              NFVI (Network Functions Virtualization Infra)      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenStack (VIM for VNFs)                                │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Nova + KVM (VNF VM compute)                       │  │   │
│  │  │  Neutron + OVS/DPDK (high-perf NFV networking)     │  │   │
│  │  │  SR-IOV (NIC passthrough for line-rate performance) │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes (VIM for CNFs)                               │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  DPDK-enabled pods │ SR-IOV device plugin          │  │   │
│  │  │  Multus CNI (multiple NICs per pod)                │  │   │
│  │  │  CPU pinning (NUMA-aware scheduling)               │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              PERFORMANCE-OPTIMIZED HOST OS                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  DPDK (Data Plane Development Kit — bypass kernel net)   │   │
│  │  HugePages (2MB/1GB pages for DPDK memory)               │  │   │
│  │  CPU isolation (isolcpus= for DPDK and VNF pinning)      │   │
│  │  NUMA topology awareness                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (COTS hardware)                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Compute nodes: Intel Xeon (DPDK-optimized) │ RAM 512GB+ │   │
│  │  NIC: Intel X710/XXV710 (SR-IOV capable) 25GbE+          │   │
│  │  BIOS: HugePages enabled │ VT-x │ VT-d (IOMMU)          │   │
│  │  (VT-d required for SR-IOV NIC passthrough)              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- OPNFV is a reference architecture for telecom NFV — it defines how OpenStack + Kubernetes work together for network functions
- OpenStack provides VMs for traditional VNFs (vFirewall, vRouter, vEPC)
- Kubernetes provides pods for cloud-native CNFs (free5GC 5G core)
- DPDK bypasses the Linux kernel network stack for line-rate packet processing
- SR-IOV passes physical NIC virtual functions directly to VMs/pods
- HugePages reduce TLB pressure for DPDK memory management
- MANO layer (OSM/ONAP) orchestrates VNF lifecycle across both OpenStack and K8s

## Use Case
Mobile operators, ISPs, and telcos replacing proprietary network hardware with software running on commodity servers. 4G/5G core networks, SD-WAN, virtual firewalls, and load balancers.

## Pros
- Eliminates expensive proprietary network hardware
- Software-defined network functions
- Carrier-grade HA and performance with DPDK + SR-IOV
- Open source reference architecture

## Cons
- Extremely complex (telco expertise + OpenStack + K8s + DPDK)
- Requires DPDK-capable hardware and careful BIOS tuning
- Long certification cycles
- Very high operational knowledge requirement