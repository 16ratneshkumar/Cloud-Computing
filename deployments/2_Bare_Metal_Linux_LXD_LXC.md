[← Back to Index](../index.md)

# DEPLOYMENT 2 — Bare Metal → Linux → LXD → LXC

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      LXC                       │
├──────────────────────────────────────────────────┤
│                      LXD                       │
├──────────────────────────────────────────────────┤
│                     Linux                      │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│               MANAGEMENT CLIENTS                                │
│  ┌──────────┐  ┌───────────────┐  ┌───────────────────────┐    │
│  │ lxc CLI  │  │  REST API     │  │  Terraform / Ansible  │    │
│  │(lxc launch│  │(HTTPS :8443) │  │  (LXD provider)       │    │
│  │ lxc list)│  │               │  │                       │    │
│  └────┬─────┘  └───────┬───────┘  └──────────┬────────────┘    │
└───────┼────────────────┼───────────────────────┼────────────────┘
        └────────────────┼───────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                    LXD DAEMON (lxd)                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  REST API Server                         │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐    │
│  │  Clustering  │  │  Profiles &  │  │  Image Manager     │    │
│  │  (Raft/dqlite│  │  Config Mgmt │  │  (simplestreams)   │    │
│  └──────────────┘  └──────────────┘  └────────────────────┘    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               STORAGE BACKENDS                          │    │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌────────┐   │    │
│  │  │ ZFS  │  │ Btrfs│  │ LVM  │  │ Ceph │  │  dir   │   │    │
│  │  └──────┘  └──────┘  └──────┘  └──────┘  └────────┘   │    │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              NETWORK BACKENDS                            │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │    │
│  │  │ bridge   │  │  OVN     │  │  macvlan / sriov     │   │    │
│  │  └──────────┘  └──────────┘  └──────────────────────┘   │    │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  creates/manages
┌────────────────────────────▼────────────────────────────────────┐
│                    LXC CONTAINERS                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Container A  │  │ Container B  │  │    Container C       │  │
│  │ Ubuntu 22.04 │  │ Alpine 3.18  │  │    Debian 12         │  │
│  │ veth0→lxdbr0 │  │ veth0→lxdbr0 │  │    veth0→OVN net    │  │
│  │ Storage: ZFS │  │ Storage: ZFS │  │    Storage: Ceph RBD │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL                                  │
│         cgroups v2 │ namespaces │ seccomp │ AppArmor            │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│      CPU │ RAM │ NVMe/SSD │ NIC (10GbE+ for clustering)        │
└─────────────────────────────────────────────────────────────────┘

CLUSTERING MODE (multiple bare metal nodes):
┌────────────────────────────────────────────────────────────────────────────┐
│  NODE 1                    │  NODE 2                    │  NODE 3          │
│  ┌──────────────────────┐  │  ┌──────────────────────┐  │  ┌───────────┐  │
│  │ LXD (cluster leader) │  │  │ LXD (cluster member) │  │  │ LXD       │  │
│  │ dqlite (Raft leader) │←─┼─→│ dqlite (follower)    │←─┼─→│ (member)  │  │
│  └──────────────────────┘  │  └──────────────────────┘  │  └───────────┘  │
│  Containers/VMs            │  Containers/VMs             │  Containers/VMs │
│  BARE METAL                │  BARE METAL                 │  BARE METAL     │
└────────────────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- LXD daemon sits above LXC and provides a full REST API, clustering via dqlite (distributed SQLite with Raft consensus), image management, and storage/network abstraction
- Clients (CLI, Terraform, REST) talk to LXD daemon
- LXD translates API calls into LXC container operations
- In cluster mode, dqlite replicates state across nodes; containers can be migrated between nodes

## Use Case
Production VPS hosting, private container clouds, developer environments needing API-driven management. Popular with Canonical-ecosystem users. Good Proxmox alternative for container-first workloads.

## Pros
- Full REST API and CLI management
- Clustering with live migration
- Multiple storage drivers (ZFS recommended)
- Both containers and VMs (QEMU) from one tool
- Image registry with simplestreams

## Cons
- Canonical controls LXD; community forked as "Incus"
- Not Kubernetes-native — separate ecosystem
- Less tooling than Docker/K8s ecosystem
- Security namespace sharing still applies