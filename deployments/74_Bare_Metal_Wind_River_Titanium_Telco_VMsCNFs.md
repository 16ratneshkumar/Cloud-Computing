[← Back to Index](../index.md)

# DEPLOYMENT 74 — Bare Metal → Wind River Titanium → Telco VMs/CNFs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                 Telco VMs/CNFs                 │
├──────────────────────────────────────────────────┤
│              Wind River Titanium               │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


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