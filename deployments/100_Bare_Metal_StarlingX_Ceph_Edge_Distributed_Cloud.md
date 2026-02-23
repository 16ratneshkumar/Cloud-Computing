[← Back to Index](../index.md)

# DEPLOYMENT 100 — Bare Metal → StarlingX + Ceph → Edge Distributed Cloud

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│             Edge Distributed Cloud             │
├──────────────────────────────────────────────────┤
│                StarlingX + Ceph                │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Telco distributed cloud — managing hundreds of edge sites (central offices, cell towers, retail) from one central cloud controller. Canadian open source project (Wind River, Intel, VMware contributions).