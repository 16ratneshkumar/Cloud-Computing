[← Back to Index](../index.md)

# DEPLOYMENT 73 — Bare Metal → OpenStack + Kubernetes → ONAP → VNFs/CNFs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   VNFs/CNFs                    │
├──────────────────────────────────────────────────┤
│                      ONAP                      │
├──────────────────────────────────────────────────┤
│             OpenStack + Kubernetes             │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


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