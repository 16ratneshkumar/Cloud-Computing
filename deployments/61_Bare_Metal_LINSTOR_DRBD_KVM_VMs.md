[← Back to Index](../index.md)

# DEPLOYMENT 61 — Bare Metal → LINSTOR (DRBD) + KVM → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│              LINSTOR (DRBD) + KVM              │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              VIRTUAL MACHINES (synchronously replicated disks)  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VM A (mission-critical database)                        │   │
│  │  disk → DRBD device (/dev/drbdN) → synchronous mirror   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM HYPERVISOR HOSTS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Host 1 (Primary)          Host 2 (Secondary)            │   │
│  │  libvirt + KVM             libvirt + KVM                 │   │
│  │  LINSTOR satellite         LINSTOR satellite             │   │
│  │  DRBD kernel module        DRBD kernel module            │   │
│  │  /dev/drbdN (primary) →→→ /dev/drbdN (secondary)        │   │
│  │  (writes sync'd before ACK)  (receives sync writes)      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINSTOR CONTROLLER + DRBD                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  LINSTOR Controller (central management daemon)          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Resource Groups (defines replication policy)      │  │   │
│  │  │  Resource Definitions (logical volumes)            │  │   │
│  │  │  Node assignments (which nodes hold replicas)      │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  LINSTOR Satellites (per node — manage local DRBD devs)  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  DRBD (kernel module) — synchronous block repl     │  │   │
│  │  │  LVM (thin pools backing DRBD devices)             │  │   │
│  │  │  ZFS (optional DRBD backend)                       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  10GbE replication network
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Nodes: CPU│RAM│NVMe (LVM thin pools)│10GbE NIC (replication)  │
│  Dedicated replication network (10GbE minimum)                  │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
High-performance synchronous block replication for KVM VMs requiring zero data loss (near-zero RPO). Financial systems, databases, and any workload where data loss is unacceptable.

## Pros
- Synchronous replication — zero data loss on failover
- Very low latency vs Ceph (no metadata overhead)
- Proven DRBD technology (LINBIT)
- Kubernetes CSI driver available
- Good for financial and database HA

## Cons
- Typically 2-3 replica limit
- Enterprise LINSTOR licensing costs
- Dedicated replication network required
- Less flexible than Ceph (block only, no object/file)
- Complex setup