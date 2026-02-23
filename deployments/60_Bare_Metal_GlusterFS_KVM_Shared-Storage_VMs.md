[← Back to Index](../index.md)

# DEPLOYMENT 60 — Bare Metal → GlusterFS + KVM → Shared-Storage VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│               Shared-Storage VMs               │
├──────────────────────────────────────────────────┤
│                GlusterFS + KVM                 │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              VIRTUAL MACHINES (live-migratable)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VM A (libvirt KVM)    VM B    VM C                      │   │
│  │  disk: qcow2 file on GlusterFS mount                     │   │
│  │  live migration → same qcow2 file accessible from both   │   │
│  │                   KVM hosts via GlusterFS                │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM HYPERVISOR HOSTS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Host 1 (KVM)               Host 2 (KVM)                 │   │
│  │  libvirt                    libvirt                       │   │
│  │  GlusterFS client (FUSE)    GlusterFS client (FUSE)      │   │
│  │  mount: /mnt/vms            mount: /mnt/vms              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  FUSE or native GlusterFS mount
┌────────────────────────────▼────────────────────────────────────┐
│              GLUSTERFS CLUSTER                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  GlusterFS Trusted Pool                                  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Brick 1 (storage node 1: /data/gluster/vm-vol)    │  │   │
│  │  │  Brick 2 (storage node 2: /data/gluster/vm-vol)    │  │   │
│  │  │  Volume type: Replicated (2-way or 3-way mirror)   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  glusterd (daemon on each storage node)                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  KVM hosts: CPU│RAM│Local SSD (for OS) │ 10GbE NIC             │
│  Storage nodes: CPU│RAM│HDDs or SSDs│10GbE NIC                 │
│  (can be converged: KVM hosts also serve as GlusterFS bricks)  │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Small to medium deployments wanting shared storage for KVM VM live migration without Ceph complexity or a SAN investment. Used in SMBs and research environments.

## Pros
- Simpler than Ceph for small deployments
- Enables KVM live migration
- No specialized storage hardware needed
- POSIX filesystem interface

## Cons
- GlusterFS reliability issues at scale historically
- Red Hat has reduced GlusterFS investment
- Performance not competitive with Ceph at scale
- Split-brain risk without proper quorum config