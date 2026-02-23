[← Back to Index](../index.md)

# DEPLOYMENT 90 — Bare Metal → Nutanix AHV + Prism → HCI Extended

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                  HCI Extended                  │
├──────────────────────────────────────────────────┤
│              Nutanix AHV + Prism               │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


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