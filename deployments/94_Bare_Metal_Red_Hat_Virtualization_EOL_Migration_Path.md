[← Back to Index](../index.md)

# DEPLOYMENT 94 — Bare Metal → Red Hat Virtualization (EOL) Migration Path

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│  Red Hat Virtualization (EOL) Migration Path   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
RHV customers need to migrate — this documents the migration path analysis rather than a live deployment.