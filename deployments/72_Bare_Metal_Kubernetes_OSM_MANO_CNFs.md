[← Back to Index](../index.md)

# DEPLOYMENT 72 — Bare Metal → Kubernetes → OSM MANO → CNFs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      CNFs                      │
├──────────────────────────────────────────────────┤
│                    OSM MANO                    │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


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