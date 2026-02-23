[← Back to Index](../index.md)

# DEPLOYMENT 58 — Bare Metal → Ceph + OpenStack → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                Ceph + OpenStack                │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              OPENSTACK WORKLOADS (using Ceph storage)           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Tenant VMs (provisioned by Nova, disks on Ceph RBD)     │   │
│  │  Object storage (Swift API → Ceph RGW backend)           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK SERVICES (Ceph-backed)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Nova → libvirt → QEMU (VMs with Ceph RBD ephemeral disk)│   │
│  │  Cinder → cinder-volume (Ceph RBD block volumes)         │   │
│  │  Glance → image service (Ceph RBD image store)           │   │
│  │  Swift API → Ceph RGW (object storage gateway)           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  librbd (direct Ceph RBD access)
┌────────────────────────────▼────────────────────────────────────┐
│              CEPH CLUSTER                                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MON (Monitors — cluster map, quorum)  [3 or 5 MONs]     │   │
│  │  MGR (Manager — metrics, modules)                        │   │
│  │  OSD (Object Storage Daemons — data storage per disk)    │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  OSD Node 1     OSD Node 2     OSD Node 3        │    │   │
│  │  │  [12× HDD]      [12× HDD]      [12× HDD]         │    │   │
│  │  │  [2× NVMe WAL]  [2× NVMe WAL]  [2× NVMe WAL]    │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  │  MDS (Metadata Server — for CephFS only)                 │   │
│  │  RGW (RADOS Gateway — S3/Swift HTTP object API)          │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  CRUSH Map (data placement algorithm)            │    │   │
│  │  │  Replication Factor: 3 (default)                 │    │   │
│  │  │  Placement Groups: distribute data across OSDs   │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  10GbE / 25GbE storage network
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CONTROLLER NODES: CPU│RAM│SSD │ 10GbE                         │
│  COMPUTE NODES:    CPU│RAM (512GB+)│NVMe│25GbE public+storage   │
│  STORAGE NODES:    CPU│RAM (64GB)│12×HDD+2×NVMe│25GbE×2        │
│  (converged: compute+storage nodes possible for smaller deploys) │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Nova configures QEMU to use Ceph RBD driver directly (`-drive format=rbd`) — VM disk I/O goes directly to Ceph, bypassing the host filesystem
- Cinder creates RBD volumes in Ceph and attaches them to VMs via librbd
- Glance stores VM images as Ceph RBD objects — Nova clones images for new VMs (fast, copy-on-write)
- Ceph CRUSH map determines data placement; OSDs self-heal by replicating data after failure
- All OpenStack services connect to Ceph via Ceph keys (cephx auth)

## Use Case
Large private cloud for telecoms, research institutions, and cloud service providers wanting no vendor storage lock-in.

## Pros
- No vendor storage lock-in
- Scales horizontally with commodity hardware
- Unified block, object, and file storage
- Self-healing replication
- Excellent OpenStack integration

## Cons
- Ceph is complex to operate
- Requires dedicated Ceph expertise
- High memory for OSDs (~1GB RAM per OSD)
- Network demands significant (separate storage network recommended)