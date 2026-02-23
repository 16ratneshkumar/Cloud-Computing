[← Back to Index](../index.md)

# DEPLOYMENT 98 — Bare Metal → XCP-ng + Ceph + CloudStack

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│           XCP-ng + Ceph + CloudStack           │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Asia-Pacific cloud hosting providers using CloudStack's AWS API compatibility with XCP-ng (free Xen) and Ceph for a cost-effective private cloud stack.