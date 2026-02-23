[← Back to Index](../index.md)

# DEPLOYMENT 71 — Bare Metal → StarlingX → Containers + VMs (Telco Edge)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│         Containers + VMs (Telco Edge)          │
├──────────────────────────────────────────────────┤
│                   StarlingX                    │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


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