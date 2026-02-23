[← Back to Index](../index.md)

# DEPLOYMENT 33 — Bare Metal → Linux → Apptainer (HPC Containers)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│           Apptainer (HPC Containers)           │
├──────────────────────────────────────────────────┤
│                     Linux                      │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              HPC JOB SCHEDULER LAYER                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  SLURM / PBS / LSF (HPC workload manager)                │   │
│  │  sbatch job.sh  →  srun apptainer exec image.sif ./app   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  runs on allocated node(s)
┌────────────────────────────▼────────────────────────────────────┐
│              APPTAINER (formerly Singularity)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  apptainer exec / run / shell / build                    │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  SIF (Singularity Image Format) — single file    │    │   │
│  │  │  Contains: OS rootfs + libraries + application   │    │   │
│  │  │  Read-only squashfs image (reproducible)         │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  CONTAINER INTERNALS                             │    │   │
│  │  │  ┌──────────────────────────────────────────┐   │    │   │
│  │  │  │  Scientific App (GROMACS / NAMD / TF)     │   │    │   │
│  │  │  │  MPI-parallel (OpenMPI / MVAPICH2)        │   │    │   │
│  │  │  │  GPU access (CUDA passthrough)            │   │    │   │
│  │  │  └──────────────────────────────────────────┘   │    │   │
│  │  │  ┌──────────────────────────────────────────┐   │    │   │
│  │  │  │  Bind mounts: home dir, scratch, MPI     │   │    │   │
│  │  │  │  Host network (no bridge — direct)       │   │    │   │
│  │  │  └──────────────────────────────────────────┘   │    │   │
│  │  └──────────────────────────────────────────────┘    │   │
│  │  No root daemon — setuid binary or user namespaces   │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (shared HPC nodes)              │
│  user namespaces │ cgroups (SLURM managed) │ IB / OmniPath net │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (HPC nodes)                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CPU (Intel Xeon / AMD EPYC — many cores)                │   │
│  │  RAM (256GB–4TB per node for in-memory workloads)        │   │
│  │  GPU (NVIDIA A100/H100 for ML/simulation)                │   │
│  │  InfiniBand (HDR 200Gb/s) or OmniPath HPC fabric        │   │
│  │  Parallel Filesystem: Lustre / GPFS / BeeGFS            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- SLURM allocates nodes to a job, then runs `apptainer exec image.sif ./simulation`
- Apptainer launches the container without a root daemon using setuid or user namespaces
- The container gets a read-only rootfs from the SIF file but bind-mounts host dirs (home, scratch, MPI lib)
- MPI processes inside the container communicate directly over InfiniBand using the host's MPI fabric
- CUDA/GPU passthrough works without modification — container sees host GPU drivers

## Use Case
Scientific computing: molecular dynamics, genomics, climate modeling, ML training on national supercomputers and university HPC clusters.

## Pros
- No root daemon — secure for multi-user HPC clusters
- MPI and GPU pass-through built-in
- Reproducible scientific environments (SIF file = complete environment)
- SLURM-compatible — works with existing HPC schedulers
- Can convert Docker images to SIF

## Cons
- Not designed for long-running microservices
- No built-in orchestration (relies on SLURM/PBS)
- Network isolation less sophisticated than Docker
- Not suitable for web services or APIs