[← Back to Index](../index.md)

# DEPLOYMENT 54 — Bare Metal → Warewulf → HPC Cluster Nodes

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│               HPC Cluster Nodes                │
├──────────────────────────────────────────────────┤
│                    Warewulf                    │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              HPC WORKLOADS (SLURM jobs)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  SLURM / PBS / LSF jobs running on compute nodes         │   │
│  │  MPI parallel jobs │ GPU simulation │ ML training        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              WAREWULF HEAD NODE                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  wwctl (Warewulf CLI)                                    │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Container images (OS rootfs) served to nodes      │  │   │
│  │  │  Node profiles (CPU/memory/GPU assignment)         │  │   │
│  │  │  Overlays (custom per-node config)                 │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  SERVICES: DHCP │ TFTP │ NFS │ NTP                       │   │
│  │  SLURM controller (schedules jobs to nodes)              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  PXE boot → initrd → diskless mount
┌────────────────────────────▼────────────────────────────────────┐
│              DISKLESS COMPUTE NODES (stateless)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1 ... Node N (could be thousands)                  │   │
│  │  Boot via PXE → load OS image over network from head node│   │
│  │  Run entirely in RAM (diskless) or with tmpfs             │   │
│  │  Shared /home and /scratch via NFS/Lustre                │   │
│  │  SLURM daemon → accepts jobs from controller             │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (HPC cluster)                  │
│  Head Node: CPU│RAM│SSD (OS+NFS)│10GbE management              │
│  Compute Nodes: CPU (many-core)│RAM│GPU│InfiniBand HDR 200Gb/s  │
│  Storage: Lustre / BeeGFS / GPFS parallel filesystem            │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Head node serves OS container images via TFTP/HTTP to compute nodes at boot
- Compute nodes PXE boot, load OS into RAM (diskless) — no per-node storage needed
- SLURM scheduler runs on head node, SLURM daemon on each compute node
- Shared storage (Lustre/NFS) provides /home and /scratch
- To update all nodes: update the container image on head node, reboot nodes

## Use Case
Supercomputing centers, national labs, university HPC clusters. Thousands of nodes managed from one head node.

## Pros
- Extremely fast reprovisioning of thousands of nodes
- No per-node storage — hardware cost savings
- Central OS image management
- Ideal for HPC batch computing

## Cons
- Not designed for persistent services or multi-tenancy
- Limited networking isolation
- Requires network boot infrastructure
- Not cloud-native