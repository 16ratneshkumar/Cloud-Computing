[← Back to Index](../index.md)

# DEPLOYMENT 93 — Bare Metal → Proxmox + Ceph → HCI (Full Stack)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                HCI (Full Stack)                │
├──────────────────────────────────────────────────┤
│                 Proxmox + Ceph                 │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS (unified HCI)                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KVM VMs (Windows, Linux, legacy apps)                   │   │
│  │  LXC containers (lightweight Linux services)             │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              PROXMOX VE (web UI port 8006)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ┌──────────────────┐  ┌────────────────────────────┐    │   │
│  │  │ VM Management    │  │  Cluster & HA Management   │    │   │
│  │  │ (KVM + LXC)      │  │  (corosync + pve-ha-mgr)   │    │   │
│  │  └──────────────────┘  └────────────────────────────┘    │   │
│  │  ┌──────────────────────────────────────────────────────┐ │   │
│  │  │  STORAGE: BUILT-IN CEPH (pveceph)                    │ │   │
│  │  │  Proxmox manages entire Ceph lifecycle:              │ │   │
│  │  │  pveceph init → pveceph createmon → pveceph createosd│ │   │
│  │  │  Ceph Dashboard embedded in Proxmox web UI          │ │   │
│  │  │  ┌────────────────────────────────────────────────┐  │ │   │
│  │  │  │  Ceph MONs │ MGRs │ OSDs │ MDS │ RGW           │  │ │   │
│  │  │  │  (all running on same Proxmox nodes)            │  │ │   │
│  │  │  └────────────────────────────────────────────────┘  │ │   │
│  │  │  RBD storage pool → VM disks (live migratable)       │ │   │
│  │  │  CephFS storage → ISO images, backups, shared        │ │   │
│  │  └──────────────────────────────────────────────────────┘ │   │
│  │  ┌──────────────────────────────────────────────────────┐ │   │
│  │  │  NETWORK: Linux bridge / VLAN-aware / OVS            │ │   │
│  │  └──────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  (Proxmox + Ceph run on same hardware)
┌────────────────────────────▼────────────────────────────────────┐
│              PROXMOX CLUSTER (3+ nodes for Ceph quorum)         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1 (Proxmox + Ceph MON/OSD)                        │   │
│  │  Node 2 (Proxmox + Ceph MON/OSD)                        │   │
│  │  Node 3 (Proxmox + Ceph MON/OSD)                        │   │
│  │  Corosync (cluster comms) ←→ dqlite (Raft, pve cluster) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL + CEPH (each node)              │
│  kvm.ko │ ceph-osd (per disk) │ ceph-mon │ ceph-mgr             │
│  LVM (local) │ ZFS (local, optional)                            │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Node 1              Node 2              Node 3 (minimum 3)    │
│  CPU 16c+            CPU 16c+            CPU 16c+              │
│  RAM 128GB+ ECC      RAM 128GB+ ECC      RAM 128GB+ ECC        │
│  NVMe: OS+KVM        NVMe: OS+KVM        NVMe: OS+KVM          │
│  HDDs: Ceph OSDs     HDDs: Ceph OSDs     HDDs: Ceph OSDs       │
│  NIC: 2×10GbE        NIC: 2×10GbE        NIC: 2×10GbE         │
│  (1 for VM traffic, 1 for Ceph replication)                    │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Proxmox integrates Ceph deployment directly via `pveceph` commands in the web UI
- All nodes run both Proxmox hypervisor (KVM/LXC) and Ceph storage daemons (MON/OSD/MGR)
- VM disks are stored as Ceph RBD volumes — enabling live migration without shared SAN
- Proxmox HA manager uses corosync for cluster communication and pve-ha-mgr for VM failover
- When a node fails: corosync detects it → HA manager restarts VMs on surviving nodes (from Ceph RBD)

## Use Case
Most popular open-source VMware alternative for SMBs and cost-conscious enterprises. 3-node minimum for full HA + Ceph — total cost of hardware only.

## Pros
- Completely free (no licensing cost)
- Integrated Ceph via web UI — no Ceph expertise needed for basics
- Live migration with Ceph RBD storage
- HA VM failover
- PBS (Proxmox Backup Server) integration

## Cons
- Ceph at scale still requires expertise
- Proxmox web UI limited for large (100+ node) clusters
- No native K8s integration (use Harvester or KubeVirt separately)
- 3-node minimum adds cost