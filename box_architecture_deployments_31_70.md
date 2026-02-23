# Complete Box-Type Architecture Diagrams
## Deployments 31–70 | Full Stack with Use Cases, Pros & Cons

---

# DEPLOYMENT 31 — Bare Metal → Docker Swarm → Containers

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MANAGEMENT CLIENTS                           │
│  ┌────────────────────┐  ┌──────────────────────────────────┐  │
│  │  docker CLI        │  │  docker stack deploy             │  │
│  │  (docker service   │  │  (Compose file → Swarm stack)    │  │
│  │   ls/scale/update) │  │                                  │  │
│  └────────────────────┘  └──────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │  Docker API (port 2377 swarm / 2376 TLS)
┌────────────────────────────▼────────────────────────────────────┐
│              DOCKER SWARM CLUSTER                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MANAGER NODES  (odd count: 1, 3, or 5 for HA)           │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Manager 1 (Leader)   Manager 2     Manager 3    │    │   │
│  │  │  ┌──────────────┐                               │    │   │
│  │  │  │  Raft        │  ← consensus for cluster state│    │   │
│  │  │  │  consensus   │                               │    │   │
│  │  │  │  (leader     │                               │    │   │
│  │  │  │   election)  │                               │    │   │
│  │  │  └──────────────┘                               │    │   │
│  │  │  Scheduler (places services on workers)         │    │   │
│  │  │  Dispatcher (sends tasks to workers)            │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  WORKER NODES                                            │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  Worker 1         Worker 2          Worker 3        │ │   │
│  │  │  ┌────────────┐   ┌────────────┐   ┌────────────┐  │ │   │
│  │  │  │ Service A  │   │ Service A  │   │ Service B  │  │ │   │
│  │  │  │ replica 1  │   │ replica 2  │   │ replica 1  │  │ │   │
│  │  │  │ (container)│   │ (container)│   │ (container)│  │ │   │
│  │  │  └────────────┘   └────────────┘   └────────────┘  │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OVERLAY NETWORK (ingress + custom)                      │   │
│  │  VXLAN-based overlay │ Routing mesh (any node → service) │   │
│  │  Encrypted overlay traffic (optional)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              DOCKER ENGINE (each node)                          │
│  dockerd │ containerd │ runc │ CNI bridge │ iptables NAT        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (each node)                     │
│  cgroups │ namespaces │ VXLAN │ iptables │ overlayfs            │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Node 1 (mgr+wkr)  │  Node 2 (worker)  │  Node 3 (worker)     │
│  CPU│RAM│SSD        │  CPU│RAM│SSD       │  CPU│RAM│SSD         │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Swarm mode is built into Docker Engine — `docker swarm init` on one node creates the cluster
- Manager nodes use Raft consensus to maintain cluster state
- Services are defined (replicas, image, ports) and the scheduler places tasks (containers) on workers
- Overlay network uses VXLAN to tunnel container traffic across nodes
- Routing mesh: any node forwards traffic to the correct service replica regardless of where it runs

## Use Case
Small to medium container orchestration for teams already using Docker. Internal apps, simple microservices, and migration step from Docker Compose to multi-node.

## Pros
- Zero extra software — built into Docker Engine
- Docker Compose-like stack files (familiar syntax)
- Simple to set up and understand
- Good for small teams not ready for Kubernetes

## Cons
- Docker Inc has deprioritized Swarm development
- No Helm/operator ecosystem
- Limited RBAC, no custom resource types
- Not recommended for new large-scale production deployments
- Kubernetes has effectively won the orchestration market

---

# DEPLOYMENT 32 — Bare Metal → Nomad → Containers + QEMU VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MANAGEMENT CLIENTS                           │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  nomad CLI       │  │  Nomad UI (web)  │  │  Terraform   │  │
│  │  nomad job run   │  │  (port 4646)     │  │  Nomad prov. │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │  RPC / HTTP API
┌────────────────────────────▼────────────────────────────────────┐
│              NOMAD SERVERS (control plane — 3 or 5)             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Raft Consensus (leader election + state replication)    │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌───────────┐  │   │
│  │  │  Job Scheduler │  │  Eval Broker   │  │  State    │  │   │
│  │  │  (bin-packing  │  │  (evaluates    │  │  Store    │  │   │
│  │  │  / spread)     │  │  job changes)  │  │  (BoltDB) │  │   │
│  │  └────────────────┘  └────────────────┘  └───────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  INTEGRATIONS                                            │   │
│  │  ┌─────────────────┐  ┌─────────────────┐              │   │
│  │  │  Consul         │  │  Vault           │              │   │
│  │  │  (service disc/ │  │  (secrets/       │              │   │
│  │  │   health check) │  │   PKI / dynamic  │              │   │
│  │  │                 │  │   credentials)   │              │   │
│  │  └─────────────────┘  └─────────────────┘              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  task assignments
┌────────────────────────────▼────────────────────────────────────┐
│              NOMAD CLIENTS (worker nodes)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │   Client Node 1                  Client Node 2           │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  TASK DRIVERS (pluggable)                          │  │   │
│  │  │  ┌──────────────┐  ┌───────────┐  ┌───────────┐   │  │   │
│  │  │  │  docker      │  │  QEMU     │  │  exec     │   │  │   │
│  │  │  │  driver      │  │  driver   │  │  (binary) │   │  │   │
│  │  │  │  (container  │  │  (full VM │  │  driver   │   │  │   │
│  │  │  │  workloads)  │  │  workload)│  │           │   │  │   │
│  │  │  └──────────────┘  └───────────┘  └───────────┘   │  │   │
│  │  │  ┌──────────────┐  ┌───────────┐                   │  │   │
│  │  │  │  java driver │  │  podman   │                   │  │   │
│  │  │  │  (JVM jobs)  │  │  driver   │                   │  │   │
│  │  │  └──────────────┘  └───────────┘                   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  RUNNING ALLOCATIONS (tasks)                       │  │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │  │   │
│  │  │  │ Container│  │  QEMU VM │  │  Java Process    │ │  │   │
│  │  │  │ (nginx)  │  │  (legacy │  │  (Kafka broker)  │ │  │   │
│  │  │  │          │  │   app)   │  │                  │ │  │   │
│  │  │  └──────────┘  └──────────┘  └──────────────────┘ │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + KVM (for QEMU tasks)                │
│  cgroups v2 │ namespaces │ KVM module (for QEMU driver)         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x for QEMU tasks) │ RAM │ Storage │ NIC               │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Nomad servers use Raft for distributed state and elect a leader
- Jobs describe desired workloads (docker/qemu/exec) — Nomad scheduler performs bin-packing onto clients
- Clients run task drivers: docker (containers), QEMU (full VMs), exec (raw binaries), java (JVM)
- Consul provides service discovery and health checking for running tasks
- Vault injects secrets directly into task environments at launch time

## Use Case
HashiCorp-ecosystem shops wanting simpler orchestration than Kubernetes for mixed workloads — containers, VMs, binaries, and JVM apps from one scheduler.

## Pros
- Handles containers, VMs, binaries, and JVM from one tool
- Much simpler operational model than Kubernetes
- Native Consul + Vault integration
- Low resource overhead
- Fast job scheduling (sub-second)

## Cons
- Smaller ecosystem than Kubernetes
- No built-in service mesh (needs Consul Connect)
- Limited storage abstractions
- Community smaller than K8s
- Container-first teams often prefer K8s

---

# DEPLOYMENT 33 — Bare Metal → Linux → Apptainer (HPC Containers)

## Box Architecture

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

---

# DEPLOYMENT 34 — Bare Metal → KVM → VMs → Kubernetes

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS (inside VMs)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes Cluster (e.g. cluster-prod-01)               │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Control Plane VMs           Worker VMs            │  │   │
│  │  │  ┌──────────────┐   ┌──────┐ ┌──────┐  ┌──────┐   │  │   │
│  │  │  │ kube-apiserver│   │wkr-1 │ │wkr-2 │  │wkr-3 │   │  │   │
│  │  │  │ etcd          │   │pods  │ │pods  │  │pods  │   │  │   │
│  │  │  │ scheduler     │   └──────┘ └──────┘  └──────┘   │  │   │
│  │  │  └──────────────┘                                   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  (Multiple isolated K8s clusters can run on same bare metal)    │
└────────────────────────────┬────────────────────────────────────┘
                             │  each K8s node = a VM
┌────────────────────────────▼────────────────────────────────────┐
│              VIRTUAL MACHINES (KVM guests)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VM: ctrl-1        VM: ctrl-2       VM: ctrl-3           │   │
│  │  (K8s ctrl plane)  (K8s ctrl plane) (K8s ctrl plane)     │   │
│  │  8vCPU / 16GB      8vCPU / 16GB     8vCPU / 16GB        │   │
│  │                                                          │   │
│  │  VM: wkr-1         VM: wkr-2        VM: wkr-3            │   │
│  │  (K8s worker)      (K8s worker)     (K8s worker)         │   │
│  │  16vCPU / 64GB     16vCPU / 64GB    16vCPU / 64GB       │   │
│  │  virtio-net        virtio-net       virtio-net           │   │
│  │  virtio-scsi       virtio-scsi      virtio-scsi          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  VM provisioning
┌────────────────────────────▼────────────────────────────────────┐
│              VM MANAGEMENT LAYER                                │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  Terraform       │  │  Ansible         │  │  libvirt /   │  │
│  │  (provision VMs) │  │  (configure K8s) │  │  virsh       │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU + KVM (hypervisor layer)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  One QEMU process per VM                                 │   │
│  │  KVM kernel module → hardware virtualization            │   │
│  │  Linux bridges / OVS → VM networking                    │   │
│  │  LVM / Ceph RBD → VM disk storage                       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + KVM MODULE (host)                   │
│  kvm.ko │ kvm-intel.ko / kvm-amd.ko │ TUN/TAP │ OVS            │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Host 1 (HV)        Host 2 (HV)        Host 3 (HV)      │   │
│  │  CPU: 64c/128t      CPU: 64c/128t      CPU: 64c/128t    │   │
│  │  RAM: 512GB         RAM: 512GB         RAM: 512GB        │   │
│  │  NVMe: 4x 4TB       NVMe: 4x 4TB      NVMe: 4x 4TB      │   │
│  │  NIC: 25GbE x2      NIC: 25GbE x2     NIC: 25GbE x2    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

MULTI-CLUSTER LAYOUT on same bare metal:
┌─────────────────────────────────────────────────────────────┐
│  BARE METAL HOST POOL                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  KVM (hypervisor)                                   │   │
│  │  ┌─────────────────┐  ┌─────────────────┐          │   │
│  │  │  K8s Cluster A  │  │  K8s Cluster B  │          │   │
│  │  │  (production)   │  │  (staging)      │          │   │
│  │  │  VMs: ctrl+wkr  │  │  VMs: ctrl+wkr  │          │   │
│  │  └─────────────────┘  └─────────────────┘          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## How It Works
- Bare metal hosts run KVM hypervisor with libvirt management
- VMs are provisioned (via Terraform/Ansible) and each VM becomes a Kubernetes node
- Multiple Kubernetes clusters share the same physical hardware via VM isolation
- Each K8s cluster has full VM isolation — blast radius contained per cluster
- VM snapshots enable quick K8s node recovery

## Use Case
Most common enterprise Kubernetes deployment. Banks, retailers, telcos running multiple K8s clusters with VM-level isolation between them on shared hardware.

## Pros
- VM isolation between K8s clusters prevents cross-cluster impact
- Familiar VM operations (snapshot, backup, clone)
- Multiple clusters on same hardware
- Easy node replacement — redeploy VM

## Cons
- VM overhead reduces container density vs bare-metal K8s
- Two layers to manage (KVM + K8s)
- Higher per-pod latency than bare-metal K8s
- Storage I/O passes through VM layer

---

# DEPLOYMENT 35 — Bare Metal → KVM → VMs → Docker/LXC (Multi-Tenant)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT APPLICATIONS (inside per-tenant VM)         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  TENANT A VM                  TENANT B VM                │   │
│  │  ┌──────────────────────┐     ┌──────────────────────┐   │   │
│  │  │  Docker containers   │     │  LXC containers       │   │   │
│  │  │  ┌────┐ ┌────┐ ┌───┐ │     │  ┌────┐ ┌────┐       │   │   │
│  │  │  │App1│ │App2│ │DB │ │     │  │Web │ │API │       │   │   │
│  │  │  └────┘ └────┘ └───┘ │     │  └────┘ └────┘       │   │   │
│  │  │  docker-compose.yml  │     │  (OS containers)      │   │   │
│  │  └──────────────────────┘     └──────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  each tenant = isolated VM
┌────────────────────────────▼────────────────────────────────────┐
│              KVM VIRTUAL MACHINES (one per tenant)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VM: tenant-A            VM: tenant-B                    │   │
│  │  Ubuntu 22.04            Debian 12                       │   │
│  │  4 vCPU / 8GB RAM        4 vCPU / 8GB RAM               │   │
│  │  Docker Engine installed  LXD installed                  │   │
│  │  veth → virbr0            veth → virbr0                  │   │
│  │  Disk: LVM thin volume   Disk: LVM thin volume          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              VM MANAGEMENT PLATFORM                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Proxmox / oVirt / OpenStack Nova / libvirt              │   │
│  │  → provisions VMs, manages billing, quota enforcement    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU + KVM                                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KVM kernel module │ QEMU processes │ LVM thin pools     │   │
│  │  Linux bridge (virbr0) │ iptables per-tenant firewall    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (HOST)                          │
│  KVM │ cgroups │ namespaces │ iptables │ TUN/TAP               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x) │ RAM │ SSD/NVMe │ 10GbE NIC                      │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Each tenant gets a dedicated VM providing hardware-level isolation
- Inside the VM, tenants run Docker or LXC for application-level container density
- VM boundaries prevent cross-tenant kernel exploits
- The platform manages VM lifecycle, billing, and networking per tenant

## Use Case
VPS and cloud hosting providers, MSPs, and multi-tenant SaaS platforms needing strong tenant isolation with container flexibility inside each tenant's boundary.

## Pros
- Strong VM-level tenant isolation
- Tenants have root in their VM
- Container density within each tenant's VM
- Simple billing model (per VM)

## Cons
- Double overhead: VM + containers
- No cross-VM container orchestration
- Higher per-tenant cost than bare container hosting
- Container sprawl hard to manage

---

# DEPLOYMENT 36 — Bare Metal → LXD Cluster → Kubernetes Nodes (as containers)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes Cluster (running inside LXD containers)      │   │
│  │  Pods → containers inside LXD container → K8s node       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s nodes are LXD containers
┌────────────────────────────▼────────────────────────────────────┐
│              LXD CONTAINERS (acting as K8s nodes)               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  lxd-k8s-ctrl-1   lxd-k8s-ctrl-2   lxd-k8s-ctrl-3      │   │
│  │  (K8s ctrl plane) (K8s ctrl plane) (K8s ctrl plane)     │   │
│  │  Ubuntu 22.04     Ubuntu 22.04     Ubuntu 22.04         │   │
│  │  own IP (LXD net) own IP           own IP               │   │
│  │                                                          │   │
│  │  lxd-k8s-wkr-1    lxd-k8s-wkr-2   lxd-k8s-wkr-3       │   │
│  │  (K8s worker)     (K8s worker)    (K8s worker)          │   │
│  │  Ubuntu 22.04     Ubuntu 22.04    Ubuntu 22.04          │   │
│  │  containerd       containerd      containerd            │   │
│  │  [nested containers inside LXD container]               │   │
│  └──────────────────────────────────────────────────────────┘   │
│  Needs: security.nesting=true on LXD containers                 │
│         security.privileged=true (or user NS remapping)         │
└────────────────────────────┬────────────────────────────────────┘
                             │  LXD cluster manages container placement
┌────────────────────────────▼────────────────────────────────────┐
│              LXD CLUSTER (across bare metal nodes)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  LXD Node 1 (BM)    LXD Node 2 (BM)    LXD Node 3 (BM)  │   │
│  │  dqlite (Raft)   ←→  dqlite (follower) ←→  dqlite       │   │
│  │  lxd daemon          lxd daemon             lxd daemon   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  Shared storage: ZFS / Ceph / NFS (for container migration)     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (each BM node)                  │
│  cgroups v2 │ namespaces │ overlayfs │ nested namespace support  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Node 1              Node 2              Node 3                │
│  CPU│RAM│NVMe        CPU│RAM│NVMe        CPU│RAM│NVMe           │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- LXD containers are configured with `security.nesting=true` to allow running nested containers (containerd inside LXD container)
- Each LXD container runs a full Ubuntu system acting as a K8s node
- K8s control plane and workers are standard kubeadm clusters — they just happen to run inside LXD containers
- LXD cluster manages container placement across physical nodes

## Use Case
High-density Kubernetes lab environments, multi-tenant dev clouds, training environments needing many K8s nodes on few physical machines.

## Pros
- Very high density — many K8s nodes per physical host
- Fast LXD container provisioning (seconds)
- Easy to scale up/down K8s nodes
- Good for labs and CI test clusters

## Cons
- Nested namespaces — complex cgroup and security config
- Not recommended for production multi-tenant (shared kernel)
- Kernel version constraints
- Some K8s features behave differently in nested environment

---

# DEPLOYMENT 37 — Bare Metal → LXD → Containers + VMs (Unified)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              UNIFIED WORKLOADS (via LXD)                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  LXC SYSTEM CONTAINERS         LXD VIRTUAL MACHINES     │   │
│  │  ┌──────────────────────┐      ┌──────────────────────┐  │   │
│  │  │ container: web-01    │      │ vm: windows-01       │  │   │
│  │  │ Ubuntu 22.04 LXC     │      │ Windows Server 2022  │  │   │
│  │  │ Nginx                │      │ (QEMU internally)    │  │   │
│  │  │ veth → lxdbr0        │      │ virtio-net           │  │   │
│  │  │ ZFS dataset          │      │ virtio-blk           │  │   │
│  │  └──────────────────────┘      │ UEFI / Secure Boot   │  │   │
│  │  ┌──────────────────────┐      └──────────────────────┘  │   │
│  │  │ container: db-01     │      ┌──────────────────────┐  │   │
│  │  │ PostgreSQL            │      │ vm: legacy-app-01    │  │   │
│  │  │                      │      │ CentOS 7 (legacy)    │  │   │
│  │  └──────────────────────┘      │ (QEMU internally)    │  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  single unified API
┌────────────────────────────▼────────────────────────────────────┐
│              LXD DAEMON (unified management)                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  REST API (:8443)                                        │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Container driver (LXC backend)                  │    │   │
│  │  │  VM driver (QEMU backend — uses qemu-kvm)        │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  │  ┌─────────────┐  ┌────────────────┐  ┌──────────────┐  │   │
│  │  │  ZFS storage│  │  OVN networking│  │  Profiles    │  │   │
│  │  │  pool        │  │  (L3 routing)  │  │  (config     │  │   │
│  │  │  (shared)    │  │                │  │   templates) │  │   │
│  │  └─────────────┘  └────────────────┘  └──────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KERNEL LAYER                                       │
│  ┌─────────────────────┐  ┌──────────────────────────────────┐  │
│  │  LXC (namespaces +  │  │  KVM + QEMU (for LXD VMs)        │  │
│  │  cgroups for conts) │  │  VMX/SVM + EPT                   │  │
│  └─────────────────────┘  └──────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V for VMs) │ RAM │ NVMe (ZFS pool) │ NIC        │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
SMBs and hosters needing both Linux containers (for efficient Linux apps) and full VMs (for Windows or legacy systems) from a single unified API without Proxmox.

## Pros
- One API, one CLI for both containers and VMs
- ZFS provides excellent storage for both types
- OVN provides advanced networking
- Snapshots and migration for both

## Cons
- LXD VM feature is less mature than Proxmox KVM
- Canonical ecosystem with community fork tension
- Not cloud-native

---

# DEPLOYMENT 38 — Bare Metal → Kubernetes → KubeVirt → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES API CLIENTS                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  kubectl apply -f vm.yaml   virtctl (KubeVirt CLI plugin)  │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES CONTROL PLANE                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kube-apiserver │ etcd │ kube-scheduler │ controllers     │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KUBEVIRT CONTROL PLANE (custom resource controllers)    │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  virt-controller (watches VM/VMI objects)          │  │   │
│  │  │  virt-api (webhook admission controller)           │  │   │
│  │  │  virt-operator (installs/upgrades KubeVirt)        │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  KUBEVIRT CRDs:                                           │   │
│  │  VirtualMachine (VM) │ VirtualMachineInstance (VMI)      │   │
│  │  VirtualMachineInstanceMigration │ DataVolume (CDI)      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  schedules VMI as a Pod
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES WORKER NODE                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CONTAINER PODS (regular)     VM PODS (KubeVirt)         │   │
│  │  ┌──────────────┐             ┌──────────────────────┐   │   │
│  │  │  App Pod A   │             │  virt-launcher Pod   │   │   │
│  │  │  (nginx)     │             │  ┌────────────────┐  │   │   │
│  │  └──────────────┘             │  │  QEMU process  │  │   │   │
│  │  ┌──────────────┐             │  │  (1 per VM)    │  │   │   │
│  │  │  App Pod B   │             │  │  KVM-backed    │  │   │   │
│  │  │  (redis)     │             │  │  VMI workload  │  │   │   │
│  │  └──────────────┘             │  └────────────────┘  │   │   │
│  │                               └──────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  virt-handler (DaemonSet — per node agent)               │   │
│  │  → manages QEMU processes on this node                   │   │
│  │  → handles live migration                                │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CDI (Containerized Data Importer)                       │   │
│  │  → imports disk images (qcow2/ISO) into PVCs             │   │
│  │  → provides DataVolume CRD for VM disk management        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              STORAGE LAYER (for VM disks)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  PVC (PersistentVolumeClaim) → DataVolume (CDI)          │   │
│  │  ┌───────────┐  ┌───────────┐  ┌──────────────────────┐ │   │
│  │  │ Ceph RBD  │  │ Longhorn  │  │ NFS / hostPath       │ │   │
│  │  │ (CSI)     │  │ (CSI)     │  │ (local dev)          │ │   │
│  │  └───────────┘  └───────────┘  └──────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (each node)                     │
│  kvm.ko │ kvm-intel/amd.ko │ QEMU process per VM               │
│  containerd (for regular pods) │ cgroups │ TUN/TAP              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V mandatory) │ RAM │ NVMe/SAN │ 25GbE NIC       │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- KubeVirt extends Kubernetes with CRDs: VirtualMachine, VirtualMachineInstance
- `kubectl apply -f vm.yaml` creates a VM object; virt-controller creates a virt-launcher Pod
- The virt-launcher Pod runs a QEMU process — the actual VM
- virt-handler (DaemonSet) manages QEMU on each node and coordinates live migration
- CDI imports disk images from external URLs/registries into PVCs for VM boot disks
- VMs get Kubernetes service discovery, RBAC, and network policies like any other workload

## Use Case
Organizations consolidating VM and container management under one Kubernetes control plane. Red Hat OpenShift Virtualization is the enterprise version. Used to migrate VMware VMs to Kubernetes.

## Pros
- Single control plane for VMs and containers
- kubectl/GitOps manages VMs
- Live migration via Kubernetes API
- Kubernetes RBAC applies to VMs
- Active CNCF project

## Cons
- Storage (CDI + PVC) setup is complex
- VM performance slightly lower than bare KVM
- Requires KubeVirt-specific expertise on top of K8s
- Debugging spans K8s + QEMU layers

---

# DEPLOYMENT 39 — Bare Metal → Kubernetes → Kata Containers → MicroVMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Pods with Kata runtime (each pod = its own VM kernel)   │   │
│  │  ┌─────────────────────┐  ┌─────────────────────────┐   │   │
│  │  │  Pod A (Kata)        │  │  Pod B (runc normal)    │   │   │
│  │  │  ┌─────────────────┐ │  │  (regular container)    │   │   │
│  │  │  │  Container App  │ │  │  ┌──────────────────┐   │   │   │
│  │  │  │  (inside VM)    │ │  │  │  Container App   │   │   │   │
│  │  │  └─────────────────┘ │  │  └──────────────────┘   │   │   │
│  │  │  Own Linux kernel!   │  │  Shared host kernel      │   │   │
│  │  └─────────────────────┘  └─────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + containerd                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  containerd (CRI)                                        │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  RuntimeClass: kata-qemu  →  kata-qemu shim      │    │   │
│  │  │  RuntimeClass: runc       →  runc (standard)     │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  kata-runtime creates VM per pod
┌────────────────────────────▼────────────────────────────────────┐
│              KATA CONTAINERS RUNTIME                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kata-runtime (shim v2)                                  │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Per-Pod MicroVM:                                │    │   │
│  │  │  ┌────────────────────────────────────────────┐  │    │   │
│  │  │  │  QEMU / Firecracker / Cloud Hypervisor     │  │    │   │
│  │  │  │  (hypervisor backend, selectable)           │  │    │   │
│  │  │  │  ┌──────────────────────────────────────┐  │  │    │   │
│  │  │  │  │  Kata agent (inside VM)              │  │  │    │   │
│  │  │  │  │  → receives OCI commands             │  │  │    │   │
│  │  │  │  │  → starts container process in VM    │  │  │    │   │
│  │  │  │  └──────────────────────────────────────┘  │  │    │   │
│  │  │  │  Minimal kernel (kata-kernel) in VM        │  │    │   │
│  │  │  └────────────────────────────────────────────┘  │    │   │
│  │  └──────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (host)                          │
│  kvm.ko │ kvm-intel/amd.ko │ QEMU/Firecracker per pod VM       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V mandatory) │ RAM │ NVMe │ NIC                 │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- `RuntimeClass: kata-qemu` tells containerd to use Kata runtime instead of runc
- Kata runtime creates a lightweight VM (QEMU/Firecracker backend) per Pod
- The Kata agent runs inside the VM, receives OCI container start commands from the host shim
- Container process runs inside the VM with its own kernel — full hardware isolation
- From Kubernetes perspective, it's just a pod — standard scheduling and networking

## Use Case
Multi-tenant Kubernetes where tenants' containers must not share a kernel (security requirement). Cloud providers offering shared K8s clusters to untrusted users.

## Pros
- Hardware isolation per pod (own kernel per pod)
- Standard Kubernetes API — no change to deployments
- Supports QEMU, Firecracker, and Cloud Hypervisor backends
- Better security than shared-kernel containers

## Cons
- Higher overhead per pod (VM boot + VM overhead)
- Slower pod startup time
- Some Kubernetes features limited (e.g., exec into container slower)
- Not all CSI drivers compatible

---

# DEPLOYMENT 40 — Bare Metal → Kubernetes → Firecracker MicroVMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              SERVERLESS / FAAS WORKLOADS                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Function A  │  Function B  │  Function C                │   │
│  │  (Python)    │  (Node.js)   │  (Go)                      │   │
│  │  Each in own MicroVM — hardware isolated                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + containerd                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RuntimeClass: kata-fc  →  Kata + Firecracker backend    │   │
│  │  OR: direct Firecracker integration (e.g., Weaveworks)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  one Firecracker VMM per pod
┌────────────────────────────▼────────────────────────────────────┐
│              FIRECRACKER MicroVM Manager                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  firecracker binary (Rust, ~1MB)                         │   │
│  │  REST API on Unix socket → management interface          │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Per-Workload MicroVM:                           │    │   │
│  │  │  ┌────────────────────────────────────────────┐  │    │   │
│  │  │  │  Minimal Linux kernel (~5.10 LTS)          │  │    │   │
│  │  │  │  rootfs (read-only, shared or per-VM)      │  │    │   │
│  │  │  │  virtio-net (TAP device)                   │  │    │   │
│  │  │  │  virtio-blk (block device)                 │  │    │   │
│  │  │  │  NO: USB, PCI hotplug, BIOS emulation      │  │    │   │
│  │  │  │  → intentionally minimal                   │  │    │   │
│  │  │  │  Boot time: ~125ms                         │  │    │   │
│  │  │  │  Overhead: <5MB per VMM process            │  │    │   │
│  │  │  └────────────────────────────────────────────┘  │    │   │
│  │  └──────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────┘   │
│  jailer process (sandboxes Firecracker — seccomp + chroot)   │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/kvm
┌────────────────────────────▼────────────────────────────────────┐
│              KVM KERNEL MODULE                                  │
│  kvm.ko + kvm-intel/amd.ko → hardware VT-x/AMD-V               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (host)                          │
│  KVM │ TUN/TAP (virtio-net) │ cgroups (jailer enforcement)      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (Intel VT-x or AMD-V + AMD64)                             │
│  RAM │ NVMe │ NIC (10GbE+)                                     │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Firecracker is a minimal VMM written in Rust, purposely removing all non-essential device emulation
- Each microVM is started by the Firecracker binary, configured via a REST API on a Unix socket
- The jailer process sandboxes Firecracker using cgroups, seccomp, and a chroot jail
- virtio-net uses a TAP device on the host for networking
- Multiple microVMs share a read-only rootfs (snapshot) — only dirty pages are per-VM
- Boot time: ~125ms; memory overhead per VMM: <5MB

## Use Case
AWS Lambda internals, self-hosted FaaS, and any scenario needing thousands of secure VM-per-request workloads with minimal overhead.

## Pros
- Sub-second startup (~125ms)
- <5MB memory overhead per VMM
- KVM hardware isolation
- Rust — memory-safe implementation
- Open source (Apache 2.0)

## Cons
- Minimal device support (intentionally limited)
- Linux guests only (x86, ARM64)
- Must build orchestration layer on top
- No storage hot-plug or USB
- Requires KVM hardware support

---

# DEPLOYMENT 41 — Bare Metal → Kubernetes → Harvester → Containers + VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT PLANE                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Harvester UI (web dashboard — port 443)                 │   │
│  │  Rancher (multi-cluster management, optional)            │   │
│  │  kubectl / Terraform Harvester provider                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              HARVESTER HCI PLATFORM                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CONTAINER WORKLOADS        VM WORKLOADS                 │   │
│  │  ┌──────────────────┐       ┌──────────────────────────┐ │   │
│  │  │  K8s Pods         │       │  KubeVirt VMs            │ │   │
│  │  │  (system +        │       │  ┌──────────────────┐    │ │   │
│  │  │   user workloads) │       │  │  Windows Server  │    │ │   │
│  │  └──────────────────┘       │  │  Ubuntu Linux    │    │ │   │
│  │                             │  └──────────────────┘    │ │   │
│  │                             └──────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  HARVESTER CORE COMPONENTS (all as K8s controllers)      │   │
│  │  ┌─────────────────┐  ┌────────────────────────────────┐ │   │
│  │  │  KubeVirt        │  │  Longhorn (distributed storage)│ │   │
│  │  │  (VM mgmt)       │  │  → block storage for VMs      │ │   │
│  │  └─────────────────┘  └────────────────────────────────┘ │   │
│  │  ┌─────────────────┐  ┌────────────────────────────────┐ │   │
│  │  │  Multus CNI      │  │  VLAN-aware networking         │ │   │
│  │  │  (multi-NIC      │  │  (VM network isolation)        │ │   │
│  │  │   for VMs)       │  │                                │ │   │
│  │  └─────────────────┘  └────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RKE2 (Rancher Kubernetes Engine 2) — K8s distribution  │   │
│  │  kube-apiserver │ etcd │ scheduler │ controllers         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL + LONGHORN DISK                 │
│  kvm.ko │ containerd │ Longhorn agents on each node             │
│  Longhorn replicates VM disk data across nodes                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Node 1 (master+worker)  Node 2 (worker)  Node 3 (worker)      │
│  CPU│RAM│NVMe (Longhorn)  CPU│RAM│NVMe     CPU│RAM│NVMe         │
│  Minimum 3 nodes for production, 1 node for dev                 │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Harvester installs as an OS (based on openSUSE Leap), bootstrapping RKE2 (K8s) automatically
- KubeVirt provides VM management within K8s
- Longhorn provides distributed block storage — VM disks are Longhorn volumes replicated across nodes
- Multus CNI allows VMs to have multiple network interfaces (e.g., management + data VLANs)
- Rancher can manage multiple Harvester clusters from a central dashboard

## Use Case
Open-source VMware alternative for organizations wanting cloud-native HCI. Used to migrate VMware workloads while gaining Kubernetes-native operations.

## Pros
- Full open-source HCI (no licensing)
- Kubernetes-native from the ground up
- Rancher integration for multi-site management
- Built-in distributed storage (Longhorn)
- Active SUSE/Rancher backing

## Cons
- Longhorn storage can be limiting at very high throughput
- Still maturing — 1.x releases
- Requires Kubernetes expertise
- Less mature Windows VM support vs VMware

---

# DEPLOYMENT 42 — Bare Metal → OpenShift → Containers + KubeVirt VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT PLANE                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenShift Console (web UI)                              │   │
│  │  oc CLI (OpenShift-extended kubectl)                     │   │
│  │  OpenShift GitOps (ArgoCD) │ OpenShift Pipelines (Tekton)│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              RED HAT OPENSHIFT PLATFORM                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OCP Control Plane (RHCOS-based)                         │   │
│  │  kube-apiserver │ etcd │ scheduler │ controllers          │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CONTAINER WORKLOADS      VM WORKLOADS (OCP Virtualization│   │
│  │  ┌──────────────────┐     ┌──────────────────────────────┐│   │
│  │  │  App Pods         │     │  KubeVirt VMIs               ││   │
│  │  │  (standard OCI)   │     │  ┌───────────────────────┐  ││   │
│  │  └──────────────────┘     │  │  Windows Server VM    │  ││   │
│  │                           │  │  RHEL 9 VM            │  ││   │
│  │                           │  │  Legacy App VM        │  ││   │
│  │                           │  └───────────────────────┘  ││   │
│  │                           └──────────────────────────────┘│   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OPENSHIFT OPERATORS (key ones)                          │   │
│  │  ┌────────────────┐  ┌──────────────┐  ┌─────────────┐  │   │
│  │  │  OCP Virt      │  │  ODF         │  │  OCP        │  │   │
│  │  │  Operator      │  │  (OpenShift  │  │  Networking │  │   │
│  │  │  (KubeVirt)    │  │   Data       │  │  (OVN-K8s)  │  │   │
│  │  │                │  │  Foundation) │  │             │  │   │
│  │  └────────────────┘  └──────────────┘  └─────────────┘  │   │
│  │  ┌────────────────┐  ┌──────────────┐                   │   │
│  │  │  Cert Manager  │  │  Monitoring  │                   │   │
│  │  │  Operator      │  │  (Prometheus │                   │   │
│  │  │                │  │  + Grafana)  │                   │   │
│  │  └────────────────┘  └──────────────┘                   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  SECURITY LAYER                                          │   │
│  │  SELinux (enforcing) │ SCC (Security Context Constraints)│   │
│  │  NetworkPolicy │ OPA/Gatekeeper │ RBAC │ Pod Security    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              RHCOS (Red Hat CoreOS — immutable OS)              │
│  CRI-O (container runtime) │ KVM │ FIPS-140 crypto              │
│  Machine Config Operator manages OS updates                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  RHEL-certified hardware │ CPU (VT-x/AMD-V) │ RAM │ NVMe │ NIC │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- OpenShift is Red Hat's enterprise Kubernetes distribution based on RHCOS (immutable OS)
- OpenShift Virtualization operator installs KubeVirt and CDI for VM management
- ODF (OpenShift Data Foundation = Rook-Ceph) provides storage for both containers and VM disks
- OVN-Kubernetes provides SDN with NetworkPolicy enforcement
- Machine Config Operator manages RHCOS node configuration declaratively

## Use Case
Enterprise organizations standardizing on Red Hat with full support contract. Financial services, healthcare, government wanting to replace VMware while gaining Kubernetes.

## Pros
- Red Hat enterprise support for VMs + containers
- SELinux hardening throughout
- GitOps-native (ArgoCD included)
- ODF provides integrated HCI storage

## Cons
- Very expensive licensing
- High complexity
- OpenShift-specific conventions
- Heavy resource requirements

---

# DEPLOYMENT 43 — Bare Metal → Talos Linux → Kubernetes → KubeVirt VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT CLIENTS                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  talosctl (Talos API client)  │  kubectl (K8s API)       │   │
│  │  NO SSH │ NO SHELL on nodes  │  virtctl (KubeVirt)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Talos API (gRPC, mTLS port 50000)
┌────────────────────────────▼────────────────────────────────────┐
│              TALOS LINUX (immutable OS)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  machined (PID 1 — Talos init, API server)               │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Everything is a static pod or system extension    │  │   │
│  │  │  No package manager │ No shell │ Read-only /usr   │  │   │
│  │  │  Configuration via talosctl apply-config           │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KUBERNETES (bootstrapped by Talos)                      │   │
│  │  kube-apiserver │ etcd │ scheduler │ kubelet              │   │
│  │  containerd (container runtime)                          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  CONTAINER PODS                                    │  │   │
│  │  │  App pods, system pods (Cilium CNI, etc.)          │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  KUBEVIRT VMs (via KubeVirt operator)              │  │   │
│  │  │  virt-launcher pods → QEMU processes               │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (Talos-specific build)                │
│  Minimal kernel │ KVM module │ CIS hardened │ no debug symbols  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V) │ RAM │ NVMe │ NIC                          │
│  PXE/iPXE boot │ No per-node manual configuration              │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Security-first Kubernetes where the OS has zero shell access. Air-gapped environments, compliance-focused deployments, and organizations treating nodes as pure API-managed infrastructure.

## Pros
- No shell/SSH — minimal attack surface
- Immutable OS — zero drift possible
- CIS-compliant by default
- API-only management (excellent for automation)
- Fast node reprovisioning via PXE

## Cons
- Steep learning curve for traditional sysadmins
- Debugging requires container-based tooling
- KubeVirt on Talos has less documentation
- Very different from conventional Linux admin

---

# DEPLOYMENT 44 — Bare Metal (Edge) → k3s → KubeVirt VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              EDGE WORKLOADS                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container Pods (lightweight edge apps)                  │   │
│  │  KubeVirt VMs (legacy edge VMs, e.g. POS terminal OS)    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              k3s + KubeVirt (edge cluster)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  k3s (single binary K8s)                                 │   │
│  │  KubeVirt operator (installed on top of k3s)             │   │
│  │  CDI (Containerized Data Importer for VM disks)          │   │
│  │  Multus / Flannel (networking)                           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (edge node)                     │
│  kvm.ko (requires VT-x/AMD-V in edge CPU) │ containerd          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Edge Node)                    │
│  Intel NUC / industrial PC / server edge appliance              │
│  CPU (must support VT-x) │ RAM 16GB+ │ NVMe 256GB+              │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Edge locations running both containers (modern apps) and VMs (legacy apps) with minimal resource footprint. Telco edge, retail, manufacturing.

## Pros
- Lightweight K8s (k3s) + VM support at edge
- Single platform for mixed workloads

## Cons
- KubeVirt adds significant resource requirements to k3s
- Limited testing of this combination
- Edge hardware must have VT-x support

---

# DEPLOYMENT 45 — Bare Metal → MicroShift → Edge Containers

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              EDGE APPLICATIONS                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container Pods (edge workloads)                         │   │
│  │  RHEL device agents │ AI/ML inference │ IoT data collect  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              MicroShift (minimal OpenShift runtime)             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Embedded OpenShift API components (~200MB RAM baseline) │   │
│  │  kube-apiserver │ controller-manager │ etcd              │   │
│  │  CRI-O (container runtime)                               │   │
│  │  OVN-Kubernetes (CNI — simplified)                       │   │
│  │  CSI (local path provisioner)                            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              RHEL 8/9 HOST OS                                   │
│  CRI-O │ KVM (optional for VM support) │ SELinux enforcing      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Edge Device)                  │
│  Industrial PC / Raspberry Pi 4+ / Edge server                  │
│  ARM64 or x86_64 │ 2GB+ RAM │ 16GB+ storage                    │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Red Hat's minimal OpenShift for edge devices — point-of-sale, factory floor, vehicle compute, and medical devices with Red Hat support.

## Pros
- ~200MB RAM baseline — very small footprint
- OpenShift API compatibility
- Works offline/disconnected
- Red Hat support

## Cons
- Very new product
- Limited production track record
- Red Hat subscription required

---

# DEPLOYMENT 46 — Bare Metal → OpenStack → KVM VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT / USER INTERFACES                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Horizon (web dashboard)  │  OpenStack CLI (openstack *)  │   │
│  │  Terraform openstack prov │  Heat (orchestration templates)│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  REST APIs (HTTPS)
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK CORE SERVICES                            │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  KEYSTONE (Identity)                                       │ │
│  │  ┌────────────────────────────────────────────────────┐   │ │
│  │  │  Authentication │ Authorization │ Service Catalog  │   │ │
│  │  │  Tokens (Fernet) │ Projects/Domains/Users          │   │ │
│  │  └────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  NOVA (Compute)                                            │ │
│  │  ┌────────────────────────────────────────────────────┐   │ │
│  │  │  nova-api  │  nova-scheduler  │  nova-conductor     │   │ │
│  │  │  nova-compute (runs on each hypervisor node)        │   │ │
│  │  │  → calls libvirt → launches QEMU/KVM VMs           │   │ │
│  │  └────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  NEUTRON (Networking)                                      │ │
│  │  ┌────────────────────────────────────────────────────┐   │ │
│  │  │  neutron-server │ neutron-agent │ ML2 plugin       │   │ │
│  │  │  OVN / OVS backend                                 │   │ │
│  │  │  Tenant networks │ Floating IPs │ Security Groups  │   │ │
│  │  │  Load Balancer (Octavia) │ VPN (Neutron VPN)       │   │ │
│  │  └────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  CINDER (Block Storage)  │  GLANCE (Image Service)         │ │
│  │  ┌──────────────────┐    │  ┌──────────────────────────┐  │ │
│  │  │  cinder-api       │    │  │  Image registry          │  │ │
│  │  │  cinder-scheduler │    │  │  (qcow2/raw/ISO images)  │  │ │
│  │  │  cinder-volume    │    │  │  Backends: Swift/Ceph/   │  │ │
│  │  │  LVM/Ceph backend │    │  │  local filesystem        │  │ │
│  │  └──────────────────┘    │  └──────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  SWIFT (Object Storage, optional)                          │ │
│  │  MANILA (Shared Filesystem, optional)                      │ │
│  │  BARBICAN (Secrets Management, optional)                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  MESSAGE BUS: RabbitMQ / Oslo.messaging                    │ │
│  │  DATABASE:    MariaDB/MySQL (Galera cluster)               │ │
│  │  CACHE:       Memcached / Redis                            │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────┘
                             │  nova-compute on each hypervisor node
┌────────────────────────────▼────────────────────────────────────┐
│              COMPUTE NODES (hypervisor hosts)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  nova-compute agent                                      │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  libvirt + QEMU/KVM                               │  │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌────────┐  │  │   │
│  │  │  │ Tenant VM A  │  │ Tenant VM B  │  │ VM N   │  │  │   │
│  │  │  │ (Ubuntu inst)│  │ (Win Server) │  │        │  │  │   │
│  │  │  └──────────────┘  └──────────────┘  └────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  neutron-agent (OVN/OVS for VM network plumbing)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (compute nodes)                 │
│  kvm.ko │ kvm-intel/amd.ko │ OVS/OVN │ libvirt                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CONTROLLER NODES (OpenStack services):                        │
│  CPU 16c │ RAM 128GB │ SSD (OS + DB + MQ) │ 10GbE×2            │
│                                                                  │
│  COMPUTE NODES (hypervisor):                                    │
│  CPU 64c/128t (EPYC/Xeon) │ RAM 512GB+ │ NVMe │ 25GbE×2        │
│                                                                  │
│  STORAGE NODES (if Ceph):                                       │
│  CPU 16c │ RAM 64GB │ 12×HDD + 2×NVMe (WAL/DB) │ 25GbE×2      │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Keystone authenticates all API requests and provides a service catalog (endpoints for each service)
- Nova receives VM launch requests, nova-scheduler picks a compute host, nova-compute uses libvirt to start QEMU/KVM
- Neutron creates virtual networks using OVN/OVS, plugging VM vNICs into tenant networks via OVS patch ports
- Cinder provisions block volumes (LVM/Ceph) and attaches them to VMs via iSCSI or virtio-blk
- All services communicate via RabbitMQ message queue; state stored in MariaDB

## Use Case
Private cloud for telecoms, research institutions, large enterprises — replacing AWS/Azure with on-premises IaaS. Tenant self-service VM provisioning with full network isolation.

## Pros
- Full IaaS private cloud — AWS-like self-service
- Multi-tenant with strong isolation
- Enormous feature set (compute, network, storage, identity, object)
- Open source with commercial support (Red Hat, Canonical, SUSE)

## Cons
- Extremely complex to deploy and operate
- High operational overhead (requires dedicated OpenStack team)
- Slow release cycles
- Overkill for small environments

---

# DEPLOYMENT 47 — Bare Metal → OpenStack Ironic → Bare Metal Nodes

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT / OPERATOR INTERFACES                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  openstack baremetal node list                           │   │
│  │  openstack baremetal node deploy <node>                  │   │
│  │  Nova (uses Ironic as compute driver)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK IRONIC                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ironic-api (REST API for bare metal mgmt)               │   │
│  │  ironic-conductor (performs actual provisioning actions) │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  PROVISIONING WORKFLOW:                            │  │   │
│  │  │  1. Power on node via IPMI/Redfish/iDRAC           │  │   │
│  │  │  2. Node PXE boots → Ironic Python Agent (IPA)     │  │   │
│  │  │  3. IPA reports hardware details back to Ironic    │  │   │
│  │  │  4. Ironic writes OS image to node disk (via IPA)  │  │   │
│  │  │  5. Node reboots into new OS                       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  DRIVERS:                                                │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  IPMI driver   │  Redfish driver   │  iDRAC driver │  │   │
│  │  │  (power/boot   │  (modern BMC API) │  (Dell DRAC)  │  │   │
│  │  │  control)      │                   │               │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  DEPLOY INTERFACES:                                      │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  direct (IPA writes image) │ iscsi │ ramdisk       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  IPMI / Redfish to BMC
┌────────────────────────────▼────────────────────────────────────┐
│              BARE METAL NODES (target hardware)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1                  Node 2               Node 3    │   │
│  │  ┌───────────────────┐   ┌──────────────┐   ┌────────┐  │   │
│  │  │  BMC (iDRAC/iLO/  │   │  BMC         │   │  BMC   │  │   │
│  │  │  IPMI/Redfish)    │   │              │   │        │  │   │
│  │  │  ← Ironic controls│   │              │   │        │  │   │
│  │  │    power + boot   │   │              │   │        │  │   │
│  │  └───────────────────┘   └──────────────┘   └────────┘  │   │
│  │  ┌───────────────────────────────────────────────────┐   │   │
│  │  │  PXE Network Boot (DHCP → TFTP → initrd/kernel)  │   │   │
│  │  │  → boots Ironic Python Agent (ramdisk)            │   │   │
│  │  │  → IPA calls home to Ironic API                   │   │   │
│  │  │  → Ironic deploys OS image to disk                │   │   │
│  │  │  → Node reboots into full OS (RHEL/Ubuntu/etc.)   │   │   │
│  │  └───────────────────────────────────────────────────┘   │   │
│  │                  BARE METAL (raw hardware)                │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Ironic stores bare metal node inventory (MAC addresses, IPMI creds, specs)
- When provisioning is requested, Ironic uses IPMI/Redfish to power on the node and set PXE boot
- Node PXE boots a minimal ramdisk containing Ironic Python Agent (IPA)
- IPA calls back to Ironic API reporting hardware, then Ironic instructs IPA to write the OS image to disk
- Node reboots into the fully provisioned OS — no hypervisor involved

## Use Case
Cloud API for bare metal servers — giving tenants raw hardware via the same OpenStack APIs as VMs. Used for HPC workloads, databases, and GPU workloads that can't tolerate VM overhead.

## Pros
- Cloud API for hardware provisioning
- Automated PXE/BMC provisioning
- No virtualization overhead for the workload
- MAAS-like bare metal capabilities within OpenStack

## Cons
- Slow provisioning (minutes per node)
- Complex BMC management
- Tenant isolation harder (physical hardware sharing)
- Node recovery requires full re-imaging

---

# DEPLOYMENT 48 — Bare Metal → OpenStack → Magnum → Kubernetes Clusters

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT INTERFACES                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  openstack coe cluster create myCluster                  │   │
│  │  kubectl (against provisioned cluster)                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK MAGNUM (Kubernetes-as-a-Service)         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  magnum-api  │  magnum-conductor                         │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  ClusterTemplate (defines: K8s ver, image, flavor) │  │   │
│  │  │  Cluster object → provisions K8s via Heat template │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  Heat Stack → Nova VMs for K8s nodes         │  │  │   │
│  │  │  │  cloud-init → runs kubeadm on each VM        │  │  │   │
│  │  │  │  Neutron → K8s node networks + LB            │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Magnum calls Nova/Neutron/Cinder
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK INFRASTRUCTURE SERVICES                  │
│  Nova (VMs for K8s nodes) │ Neutron (K8s networking)            │
│  Cinder (persistent volumes) │ Octavia (K8s LoadBalancer)       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM COMPUTE NODES + LINUX KERNEL                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  K8s Control Plane VMs    K8s Worker VMs                 │   │
│  │  (provisioned by Magnum)  (provisioned by Magnum)        │   │
│  │  ┌────────────────┐       ┌────────────────────────────┐ │   │
│  │  │ kube-apiserver │       │  kubelet │ containerd       │ │   │
│  │  │ etcd │ scheduler│      │  K8s pods (tenant apps)    │ │   │
│  │  └────────────────┘       └────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Compute nodes │ Storage nodes │ Network nodes │ Controllers    │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Kubernetes-as-a-Service for tenants of an OpenStack private cloud — self-service K8s cluster provisioning within an existing OpenStack environment.

## Pros
- K8s clusters as first-class OpenStack resources
- Self-service for tenants
- Integrates with OpenStack auth, networking, storage

## Cons
- Complex (OpenStack + Magnum + K8s)
- Slow cluster provisioning
- Magnum has limited development momentum
- Many failure points

---

# DEPLOYMENT 49 — Bare Metal → OpenStack → VMs → Kubernetes

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes Pods (application workloads)                 │   │
│  │  PVCs backed by Cinder volumes (OpenStack storage)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (running inside OpenStack VMs)          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Control Plane VMs (Nova instances)  Worker VMs          │   │
│  │  ┌─────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │ kube-apiserver   │  │  kubelet │ containerd        │  │   │
│  │  │ etcd │ scheduler│  │  K8s pods                    │  │   │
│  │  └─────────────────┘  └──────────────────────────────┘  │   │
│  │  cloud-provider-openstack (integrates K8s with OpenStack)│   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Cinder CSI driver → Cinder volumes as PVCs       │  │   │
│  │  │  Octavia CCM → OpenStack LB as K8s LoadBalancer   │  │   │
│  │  │  Neutron CNI → Kuryr (K8s pods on Neutron nets)   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s nodes = Nova VMs
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK NOVA (compute)                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Nova VMs (K8s nodes) with appropriate flavors           │   │
│  │  Neutron networks for K8s node connectivity              │   │
│  │  Cinder volumes for K8s persistent storage               │   │
│  │  Octavia load balancers for K8s service endpoints        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM COMPUTE + LINUX KERNEL                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  OpenStack compute/storage/network/controller nodes             │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Traditional enterprise cloud pattern: OpenStack provides IaaS, Kubernetes runs on top. Used by telcos/enterprises that adopted OpenStack first and later added K8s.

## Pros
- Leverage existing OpenStack investment
- VM isolation for K8s nodes
- OpenStack services (Cinder/Octavia) integrate with K8s

## Cons
- Two complex systems to operate
- Double overhead (OpenStack VM + K8s pod)
- Most orgs are simplifying away from this pattern

---

# DEPLOYMENT 50 — Bare Metal → Kubernetes → OpenStack Control Plane (Containerized)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT CLIENTS                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  openstack CLI  │  Horizon  │  kubectl  │  Helm / ArgoCD │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (base layer for OpenStack)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OPENSTACK CONTROL PLANE PODS (via OpenStack-Helm / OSO) │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │  keystone    │  │  nova        │  │  neutron     │   │   │
│  │  │  pod(s)      │  │  pod(s)      │  │  pod(s)      │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │  cinder      │  │  glance      │  │  horizon     │   │   │
│  │  │  pod(s)      │  │  pod(s)      │  │  pod(s)      │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Infrastructure pods:                            │    │   │
│  │  │  MariaDB (Galera)  │  RabbitMQ  │  Memcached    │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  │  K8s manages: restarts, scaling, upgrades of OpenStack   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  nova-compute on PHYSICAL compute nodes
┌────────────────────────────▼────────────────────────────────────┐
│              BARE METAL COMPUTE NODES                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  nova-compute agent (runs directly, not in K8s)          │   │
│  │  libvirt + KVM → launches actual tenant VMs              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │  Tenant VM A │  │  Tenant VM B │  │  Tenant VM C │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  │  OVN agents (neutron networking for VMs)                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  K8s CONTROL NODES (run OpenStack pods):                        │
│  CPU 32c │ RAM 256GB │ NVMe │ 25GbE NIC                        │
│                                                                  │
│  COMPUTE NODES (run tenant KVM VMs):                            │
│  CPU 64c/128t │ RAM 512GB+ │ NVMe │ 25GbE NIC                  │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Kubernetes cluster hosts all OpenStack control plane services as pods (via OpenStack-Helm charts or OpenStack Operator)
- K8s manages OpenStack pods — if nova-api crashes, K8s restarts it automatically
- MariaDB, RabbitMQ, Memcached also run as pods with K8s StatefulSets
- nova-compute still runs directly on bare metal compute nodes (not in K8s) for direct KVM access
- OpenStack API calls flow: tenant → OpenStack pods (in K8s) → nova-compute on bare metal → KVM VMs

## Use Case
Advanced telco and cloud providers wanting GitOps-managed OpenStack with K8s-native self-healing control plane. AT&T, Verizon-scale deployments.

## Pros
- OpenStack upgrades become Helm chart upgrades
- K8s self-heals OpenStack control plane pods
- GitOps-friendly (ArgoCD manages OpenStack)
- Better control plane resource utilization

## Cons
- Extreme complexity (K8s + OpenStack expertise required)
- Debugging spans multiple abstraction layers
- Very limited public documentation and case studies
- Long initial deployment timeline

---

# DEPLOYMENT 51 — Bare Metal → MAAS → OpenStack / Kubernetes / KVM Hosts

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              OPERATOR / AUTOMATION CLIENTS                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  maas CLI (maas admin machines)                          │   │
│  │  MAAS Web UI (port 5240)                                 │   │
│  │  Juju (service deployment on MAAS-managed nodes)         │   │
│  │  Terraform MAAS provider                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  MAAS API (REST)
┌────────────────────────────▼────────────────────────────────────┐
│              MAAS (Metal-as-a-Service) PLATFORM                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MAAS Region Controller                                  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  API server  │  PostgreSQL DB  │  DNS (bind9)      │  │   │
│  │  │  Machine state machine (NEW→COMMISSIONING→READY    │  │   │
│  │  │  →ALLOCATED→DEPLOYING→DEPLOYED)                    │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  MAAS Rack Controller (per rack / L2 broadcast domain)   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  DHCP server │ TFTP server │ HTTP image cache      │  │   │
│  │  │  BMC relay   │ proxy DHCP  │                       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  COMMISSIONING FLOW:                                     │   │
│  │  1. Node powers on → PXE → commissioning OS (Ubuntu)    │   │
│  │  2. MAAS agent collects: CPU, RAM, disk, NIC details     │   │
│  │  3. Node enters READY state in MAAS inventory            │   │
│  │  DEPLOYMENT FLOW:                                        │   │
│  │  4. Operator allocates + deploys node (target OS)        │   │
│  │  5. Node PXE boots deployer image                        │   │
│  │  6. Target OS written to disk via curtin                 │   │
│  │  7. Node reboots into target OS (Ubuntu/RHEL/etc.)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  IPMI/Redfish power control + PXE
┌────────────────────────────▼────────────────────────────────────┐
│              BARE METAL NODE POOL                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  DEPLOYED AS: OpenStack Controller/Compute/Storage Nodes │   │
│  │            OR: Kubernetes Control Plane / Worker Nodes   │   │
│  │            OR: Standalone KVM Hypervisor Hosts           │   │
│  │            OR: Any Ubuntu / RHEL / Custom workload       │   │
│  │  All managed via BMC (iDRAC/iLO/IPMI/Redfish)           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- MAAS discovers nodes via PXE / BMC and commissions them (collects hardware inventory)
- Operator allocates nodes to workloads (OpenStack, K8s, KVM) and deploys target OS
- MAAS handles PXE boot, DNS, DHCP for all nodes in the physical network
- Juju can deploy complex services (like Charmed OpenStack) on MAAS-provisioned nodes
- All subsequent infrastructure (OpenStack, K8s) runs on nodes MAAS has provisioned

## Use Case
Ubuntu/Canonical ecosystem bare metal automation. Foundation for Charmed OpenStack and Charmed Kubernetes deployments.

## Pros
- Full hardware lifecycle management
- API-driven node provisioning
- IPMI/Redfish/iDRAC/iLO support
- Integrates with Juju for service deployment

## Cons
- Ubuntu/Canonical focused
- Requires MAAS region/rack controllers
- Less flexible for non-Ubuntu stacks

---

# DEPLOYMENT 52 — Bare Metal → Kubernetes → Metal3 → Bare Metal Nodes

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              GITOPS / AUTOMATION CLIENTS                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kubectl apply -f baremetalhost.yaml                     │   │
│  │  ClusterAPI (CAPI) controller                            │   │
│  │  ArgoCD / FluxCD (GitOps)                                │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Kubernetes CRDs
┌────────────────────────────▼────────────────────────────────────┐
│              MANAGEMENT KUBERNETES CLUSTER                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Metal3 (baremetal-operator)                             │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  BareMetalHost CRD (represents a physical server)  │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  spec:                                       │  │  │   │
│  │  │  │    bmc:                                      │  │  │   │
│  │  │  │      address: redfish://10.0.0.1             │  │  │   │
│  │  │  │    image:                                    │  │  │   │
│  │  │  │      url: https://images/rhcos.qcow2         │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Ironic (runs as pods in K8s)                      │  │   │
│  │  │  ironic-api pod │ ironic-conductor pod             │  │   │
│  │  │  ironic-inspector pod                              │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Cluster API (CAPI) + CAPM3 provider              │  │   │
│  │  │  → provisions entire K8s clusters on bare metal   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Ironic → IPMI/Redfish BMC
┌────────────────────────────▼────────────────────────────────────┐
│              BARE METAL NODE POOL (target hardware)             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1                Node 2              Node 3        │   │
│  │  BMC: Redfish         BMC: Redfish         BMC: Redfish  │   │
│  │  ← Metal3/Ironic      ← Metal3/Ironic      ← Metal3      │   │
│  │    controls power,       controls              controls  │   │
│  │    boot order,           power+boot           power+boot │   │
│  │    OS image deploy       OS image             OS image   │   │
│  │                                                          │   │
│  │  After provisioning → become K8s worker nodes           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Metal3 introduces `BareMetalHost` CRD — physical servers become Kubernetes objects
- baremetal-operator reconciles BareMetalHost state — if `spec.online: true`, it powers on the node
- Ironic (running as K8s pods) handles the actual provisioning: BMC control, PXE boot, image deploy
- Cluster API (CAPI) with CAPM3 provider creates K8s clusters by provisioning BareMetalHosts
- Entire bare metal infrastructure is managed as YAML in Git — true GitOps for hardware

## Use Case
GitOps-driven bare metal Kubernetes provisioning. Foundation of OpenShift on bare metal and modern K8s cluster lifecycle management.

## Pros
- Bare metal as code (YAML BareMetalHost resources)
- GitOps-friendly
- Cluster API integration for automated cluster lifecycle
- Active CNCF community

## Cons
- Complex setup (Metal3 + Ironic + CAPI)
- Slow provisioning (~5-10 minutes per node)
- Requires out-of-band management (Redfish/IPMI)
- Limited advanced networking configuration

---

# DEPLOYMENT 53 — Bare Metal → Ironic Standalone → Kubernetes Nodes

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES CLUSTER (after provisioning)            │
│  kube-apiserver │ kubelet │ containerd │ Pods                   │
└────────────────────────────┬────────────────────────────────────┘
                             │  nodes provisioned by Ironic
┌────────────────────────────▼────────────────────────────────────┐
│              IRONIC STANDALONE (no full OpenStack)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ironic-api │ ironic-conductor │ ironic-inspector        │   │
│  │  (runs without Nova, Neutron, Keystone — minimal mode)   │   │
│  │  Standalone DHCP/TFTP for PXE boot                       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Physical servers with BMC (IPMI/Redfish) │ PXE-capable NICs   │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Lightweight bare metal provisioning without full OpenStack complexity — just use Ironic to automate hardware for Kubernetes clusters.

---

# DEPLOYMENT 54 — Bare Metal → Warewulf → HPC Cluster Nodes

## Box Architecture

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

---

# DEPLOYMENT 55 — Bare Metal → Apache CloudStack → KVM/Xen/VMware → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT / ADMIN INTERFACES                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CloudStack UI (web)  │  cloudmonkey CLI                 │   │
│  │  AWS EC2-compatible API (CloudStack implements EC2 API)  │   │
│  │  Terraform CloudStack provider                           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  HTTP/HTTPS API
┌────────────────────────────▼────────────────────────────────────┐
│              APACHE CLOUDSTACK MANAGEMENT SERVER                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CloudStack Management Server (Java-based)               │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Zone / Pod / Cluster / Host hierarchy             │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  Zone (data center)                          │  │  │   │
│  │  │  │  └── Pod (server rack group)                 │  │  │   │
│  │  │  │       └── Cluster (same hypervisor type)     │  │  │   │
│  │  │  │            └── Hosts (hypervisor nodes)      │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  │  VM Orchestration │ Network Mgmt │ Storage Mgmt    │  │   │
│  │  │  Account / Domain hierarchy (multi-tenancy)        │  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  MySQL (management DB) │ NFS / Ceph (secondary storage)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  CloudStack agent on each host
┌────────────────────────────▼────────────────────────────────────┐
│              HYPERVISOR HOSTS                                   │
│  ┌────────────────────┐  ┌────────────────┐  ┌──────────────┐  │
│  │  KVM Host          │  │  XCP-ng (Xen)  │  │  VMware ESXi │  │
│  │  CloudStack agent  │  │  CloudStack    │  │  Host        │  │
│  │  libvirt           │  │  agent         │  │              │  │
│  │  ┌──────────────┐  │  │  ┌──────────┐  │  │  ┌────────┐ │  │
│  │  │ Tenant VMs   │  │  │  │ VMs      │  │  │  │ VMs    │ │  │
│  │  └──────────────┘  │  │  └──────────┘  │  │  └────────┘ │  │
│  └────────────────────┘  └────────────────┘  └──────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Hypervisor hosts │ NFS/Ceph storage │ management network      │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- CloudStack Management Server is a single Java application (simpler than OpenStack's ~30 services)
- Zone/Pod/Cluster/Host hierarchy maps to data center physical structure
- CloudStack agent on each host receives commands from management server
- Supports KVM, XCP-ng/Xen, and VMware ESXi as hypervisors
- AWS EC2 API compatibility allows AWS tooling (CLI/SDK) to work against CloudStack

## Use Case
Hosting providers building public/private clouds wanting AWS API compatibility. Simpler to operate than OpenStack. Popular in Asia-Pacific cloud market.

## Pros
- Simpler deployment than OpenStack (single Java process)
- Supports multiple hypervisors from one management plane
- AWS EC2-compatible API
- Strong multi-tenancy with account/domain hierarchy

## Cons
- Smaller community than OpenStack/Kubernetes
- Less modular architecture
- Limited modern cloud-native features (no K8s integration built-in)
- Fewer public cloud providers using it

---

# DEPLOYMENT 56 — Bare Metal → OpenNebula → KVM / LXD / VMware → VMs + Containers

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT CLIENTS                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Sunstone (web UI)  │  onevm CLI  │  Terraform provider  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENNEBULA FRONT-END                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  oned (OpenNebula daemon — core)                         │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Scheduler │ ACL Manager │ Image/Network Manager   │  │   │
│  │  │  VM lifecycle │ Cluster management                 │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  SQLite / MySQL (state DB)                               │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  ONE driver to each host
┌────────────────────────────▼────────────────────────────────────┐
│              HYPERVISOR HOSTS (via SSH-based driver)            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  KVM Host        │  │  LXD Host        │  │  VMware ESXi │  │
│  │  (kvm driver)    │  │  (lxd driver)    │  │  (vmware drv)│  │
│  │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────┐  │  │
│  │  │ VMs        │  │  │  │ Containers │  │  │  │ VMs    │  │  │
│  │  │ (QEMU/KVM) │  │  │  │ (LXC)     │  │  │  │        │  │  │
│  │  └────────────┘  │  │  └────────────┘  │  │  └────────┘  │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Hypervisor nodes │ Shared storage (NFS/Ceph/iSCSI) │ Network  │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Lightweight private cloud — simpler than OpenStack. Used by European research clouds, universities, and mid-size enterprises.

## Pros
- Much simpler than OpenStack to deploy/operate
- Multi-hypervisor (KVM, LXD, VMware, Firecracker)
- Flexible VM templates
- Good edge cloud support

## Cons
- Smaller community than OpenStack
- Less feature-rich for very large deployments
- Limited RBAC compared to OpenStack/K8s

---

# DEPLOYMENT 57 — Bare Metal → Eucalyptus → KVM/Xen → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              AWS-COMPATIBLE PRIVATE CLOUD                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  AWS CLI / Boto3 SDK (works against Eucalyptus EC2 API)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  EC2-compatible API
┌────────────────────────────▼────────────────────────────────────┐
│              EUCALYPTUS CLOUD CONTROLLER                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CLC (Cloud Controller) │ Walrus (S3-compat storage)     │   │
│  │  CC (Cluster Controller) │ SC (Storage Controller)       │   │
│  │  NC (Node Controller) on each hypervisor node            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM / XEN HYPERVISOR NODES                         │
│  VMs (launched by NC agent via libvirt/KVM or Xen)             │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Historical: AWS-compatible private cloud. **Status: Effectively unmaintained — do not use for new deployments.**

---

# DEPLOYMENT 58 — Bare Metal → Ceph + OpenStack → VMs

## Box Architecture

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

---

# DEPLOYMENT 59 — Bare Metal → Ceph + Kubernetes (Rook) + KubeVirt → VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container Pods (PVCs → Ceph RBD / CephFS)               │   │
│  │  KubeVirt VMs (DataVolumes → Ceph RBD)                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES                                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ROOK OPERATOR (manages Ceph as K8s resources)           │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  CephCluster CRD → creates/manages Ceph cluster    │  │   │
│  │  │  CephBlockPool CRD → Ceph RBD pool for PVCs        │  │   │
│  │  │  CephFilesystem CRD → CephFS for shared PVCs       │  │   │
│  │  │  CephObjectStore CRD → S3/Swift API                │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Rook-Ceph CSI Driver (provisioner + attacher)     │  │   │
│  │  │  → dynamic PVC provisioning from Ceph RBD          │  │   │
│  │  │  → ReadWriteMany PVCs via CephFS                   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  KUBEVIRT OPERATOR                                 │  │   │
│  │  │  VMs with DataVolumes backed by Ceph RBD PVCs      │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Rook deploys Ceph pods on nodes
┌────────────────────────────▼────────────────────────────────────┐
│              CEPH CLUSTER (managed by Rook in K8s)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ceph-mon pods (3+) │ ceph-mgr pods │ ceph-osd pods      │   │
│  │  (each OSD pod = one physical disk on a node)            │   │
│  │  ceph-mds pods (CephFS) │ ceph-rgw pods (object)         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (each node)                     │
│  kvm.ko │ containerd │ direct disk access for OSD pods          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Converged nodes (K8s + Ceph on same hardware):                 │
│  CPU 32c+ │ RAM 256GB+ │ NVMe (for K8s) + HDDs (for Ceph OSDs) │
│  25GbE NIC (shared cluster + storage traffic)                   │
│  Minimum 3 nodes for Ceph quorum                                │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Rook Operator runs inside Kubernetes and manages Ceph cluster lifecycle via CRDs
- Rook deploys ceph-mon, ceph-mgr, ceph-osd pods on K8s nodes — each OSD pod claims a raw disk
- Rook-Ceph CSI driver enables dynamic PVC provisioning: `kubectl create pvc` → RBD volume in Ceph
- KubeVirt DataVolumes use Ceph RBD PVCs as VM boot disks — live migration moves VM + Ceph handles replication
- This creates a fully Kubernetes-native HCI stack (Harvester uses this architecture)

## Use Case
Cloud-native HCI — open-source alternative to Nutanix/vSAN. Organizations wanting one K8s control plane for VMs, containers, and storage.

## Pros
- Fully K8s-native — Ceph managed as K8s CRDs
- Dynamic storage provisioning for both containers and VMs
- Self-healing distributed storage
- No OpenStack dependency

## Cons
- Rook-Ceph adds K8s operational complexity
- Ceph tuning knowledge still required
- Minimum 3 nodes for quorum
- VM disk latency from RBD can be higher than local NVMe

---

# DEPLOYMENT 60 — Bare Metal → GlusterFS + KVM → Shared-Storage VMs

## Box Architecture

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

---

# DEPLOYMENT 61 — Bare Metal → LINSTOR (DRBD) + KVM → VMs

## Box Architecture

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

---

# DEPLOYMENT 62 — Bare Metal → Firecracker → MicroVMs (Standalone)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS (per-MicroVM isolation)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Lambda Function A  │  Lambda Function B  │  Function C  │   │
│  │  (Python 3.11)      │  (Node.js 20)       │  (Go 1.21)   │   │
│  │  Own Linux kernel   │  Own Linux kernel   │  Own kernel  │   │
│  │  Own rootfs         │  Own rootfs         │  Own rootfs  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  one Firecracker process per MicroVM
┌────────────────────────────▼────────────────────────────────────┐
│              FIRECRACKER VMM LAYER                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  jailer (security sandbox per Firecracker process)       │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  chroot (isolates Firecracker binary + config)     │  │   │
│  │  │  cgroup (limits CPU/mem for this MicroVM)          │  │   │
│  │  │  seccomp (syscall filter — strict allowlist)       │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  firecracker binary (Rust VMM ~1MB)          │  │  │   │
│  │  │  │  REST API on Unix socket (config + control)  │  │  │   │
│  │  │  │  ┌────────────────────────────────────────┐  │  │  │   │
│  │  │  │  │  MicroVM:                              │  │  │  │   │
│  │  │  │  │  virtio-net (TAP device)               │  │  │  │   │
│  │  │  │  │  virtio-blk (block device or mem)      │  │  │  │   │
│  │  │  │  │  Minimal Linux kernel (5.10 LTS)       │  │  │  │   │
│  │  │  │  │  rootfs (snapshot-based, copy-on-write)│  │  │  │   │
│  │  │  │  │  MMIO devices ONLY (no PCI)            │  │  │  │   │
│  │  │  │  └────────────────────────────────────────┘  │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  [Thousands of Firecracker processes per host]           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/kvm (ioctls)
┌────────────────────────────▼────────────────────────────────────┐
│              KVM KERNEL MODULE                                  │
│  kvm.ko + kvm-intel.ko / kvm-amd.ko                            │
│  EPT (Extended Page Tables) for guest memory isolation          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (host)                          │
│  cgroups v2 (per jailer) │ TUN/TAP │ seccomp │ iptables        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU: Intel VT-x or AMD-V (mandatory) + AMD64                  │
│  RAM: High density (1000 MicroVMs × 128MB = 128GB+)             │
│  NVMe: Fast local storage for rootfs snapshots                  │
│  NIC: 10GbE+ (many TAP devices, one per MicroVM)               │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- jailer creates a chroot + cgroup + seccomp sandbox for each Firecracker process
- Firecracker is configured via REST API calls on its Unix socket before VM boot
- VM gets minimal Linux kernel + rootfs (read-only snapshot, copy-on-write per VM)
- virtio-net uses a host TAP device for networking
- Boot: ~125ms; Memory per VMM: <5MB; allows thousands per host
- AWS Lambda runs each function invocation in a Firecracker MicroVM — this is Firecracker's primary production deployment

## Use Case
Serverless/FaaS platforms, multi-tenant code execution, and any scenario needing fast startup + strong isolation at scale. AWS Lambda, AWS Fargate internals.

## Pros
- Sub-second startup (~125ms cold start)
- <5MB VMM overhead per MicroVM
- KVM hardware isolation
- Open source (Apache 2.0, by AWS)
- Rust — memory-safe

## Cons
- Minimal device support (intentional)
- Linux guests only (x86/ARM64)
- No GPU passthrough
- Must build orchestration layer on top
- Requires KVM hardware support

---

# DEPLOYMENT 63 — Bare Metal → Cloud Hypervisor → MicroVMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              CLOUD-NATIVE VM WORKLOADS                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Linux guest VMs (more device support than Firecracker)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              CLOUD HYPERVISOR (Intel + Microsoft project)       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  cloud-hypervisor binary (Rust, ~2MB)                    │   │
│  │  REST API on Unix socket                                 │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  MicroVM (more devices than Firecracker):          │  │   │
│  │  │  virtio-net │ virtio-blk │ virtio-fs               │  │   │
│  │  │  virtio-mem (hot-plug memory) │ vDPA               │  │   │
│  │  │  PCI hotplug (devices can be added at runtime)     │  │   │
│  │  │  Minimal Linux kernel                              │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/kvm
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Intel VT-x or AMD-V │ RAM │ NVMe │ NIC                        │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Cloud infrastructure experiments, research into minimal hypervisors, and scenarios needing more device support than Firecracker while keeping minimal overhead.

## Pros
- More device support than Firecracker (virtio-fs, vDPA, hot-plug)
- Open source (Apache 2.0)
- Written in Rust — memory safe
- Active Intel + Microsoft development

## Cons
- Less production adoption than Firecracker
- Smaller community
- Fewer integrations with orchestration platforms
- Still maturing

---

# DEPLOYMENT 64 — Bare Metal → Kubernetes → OpenFaaS → Firecracker → Functions

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              DEVELOPER / FUNCTION CLIENTS                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  faas-cli deploy --image myfunction:latest               │   │
│  │  curl https://gateway.example.com/function/myfunction    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENFAAS COMPONENTS (K8s Deployments)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  faas-netes (K8s backend for OpenFaaS)                   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  API Gateway (faas-gateway pod)                    │  │   │
│  │  │  → receives HTTP invocations                       │  │   │
│  │  │  → routes to function pods                        │  │   │
│  │  │  → auto-scaling (scale to zero + scale up)        │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Prometheus (auto-scaling metrics)                 │  │   │
│  │  │  NATS (async function invocation queue)            │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  each function = a Pod with Firecracker runtime
┌────────────────────────────▼────────────────────────────────────┐
│              FUNCTION PODS (Firecracker-backed via Kata/runtime)│
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Function Pod A     Function Pod B     Function Pod N    │   │
│  │  RuntimeClass:      RuntimeClass:      RuntimeClass:     │   │
│  │  kata-firecracker   kata-firecracker   kata-firecracker  │   │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐  │   │
│  │  │ Firecracker  │   │ Firecracker  │   │ Firecracker  │  │   │
│  │  │ MicroVM      │   │ MicroVM      │   │ MicroVM      │  │   │
│  │  │ (Python fn)  │   │ (Node.js fn) │   │ (Go fn)      │  │   │
│  │  └──────────────┘   └──────────────┘   └──────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + KVM + LINUX KERNEL                    │
│  containerd │ Kata runtime (kata-fc) │ KVM module              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V) │ RAM │ NVMe │ 10GbE NIC                    │
└─────────────────────────────────────────────────────────────────┘

INVOCATION FLOW:
developer → HTTP → Gateway pod → Route → Function pod
→ Kata shim → Firecracker MicroVM → function executes
→ return → Gateway → developer
```

## How It Works
- OpenFaaS gateway receives HTTP function invocations
- faas-netes deploys function workloads as K8s pods with `RuntimeClass: kata-firecracker`
- Kata runtime uses Firecracker as its VM backend — each pod runs in a MicroVM
- Prometheus metrics drive HPA (Horizontal Pod Autoscaler) for scale-to-zero and scale-up
- NATS provides async invocation queuing for high-throughput scenarios

## Use Case
Self-hosted serverless platform for organizations with data residency requirements or cost savings vs AWS Lambda at scale.

## Pros
- True serverless on bare metal
- Strong Firecracker VM isolation per function
- Cost savings vs public cloud FaaS at scale
- Data residency compliance

## Cons
- Complex multi-layer stack (K8s + OpenFaaS + Kata + Firecracker)
- Cold start latency worse than warm containers
- High operational overhead for "serverless"
- Not as battle-tested as AWS Lambda

---

# DEPLOYMENT 65 — Bare Metal → Kubernetes → Knative → KubeVirt VMs

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              EVENT-DRIVEN VM WORKLOADS                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Legacy App VM (scales to zero when idle)                │   │
│  │  KubeVirt VMI triggered by Knative event                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  (experimental)
┌────────────────────────────▼────────────────────────────────────┐
│              KNATIVE SERVING + EVENTING                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Knative Serving → scale-to-zero for K8s workloads       │   │
│  │  Knative Eventing → CloudEvents trigger VM start         │   │
│  │  (KubeVirt integration: experimental/community built)    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + KUBEVIRT                              │
│  kube-apiserver │ KubeVirt operator │ virt-launcher pods        │
│  QEMU processes (one per VM)                                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Serverless scaling for VM workloads — experimental pattern for legacy apps that can't be containerized but need event-driven provisioning.

## Pros
- Scale-to-zero for VM workloads
- Event-driven provisioning
- Kubernetes-native

## Cons
- Highly experimental — not production-ready
- VM cold starts are slow (seconds to minutes)
- Limited documentation and community examples
- Knative + KubeVirt integration is not well-maintained

---

# DEPLOYMENT 66 — Bare Metal → Xen → MirageOS Unikernels

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              UNIKERNEL APPLICATIONS                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Unikernel A         Unikernel B         Unikernel C      │   │
│  │  (web server)        (DNS resolver)      (VPN endpoint)   │   │
│  │  Compiled OCaml      Compiled OCaml      Compiled OCaml   │   │
│  │  + only required OS  + only required OS  + only OS libs   │   │
│  │  libs linked in      libs linked in      linked in        │   │
│  │  ~2MB image          ~1MB image          ~3MB image       │   │
│  │  No shell │ No package mgr │ No unnecessary syscalls      │   │
│  └──────────────────────────────────────────────────────────┘   │
│  Each unikernel = a complete single-purpose bootable image      │
└────────────────────────────┬────────────────────────────────────┘
                             │  each is a Xen DomU guest
┌────────────────────────────▼────────────────────────────────────┐
│              XEN HYPERVISOR + Dom0                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Dom0 (Linux) — creates and manages unikernel DomUs      │   │
│  │  xl create unikernel.cfg                                 │   │
│  │  xl console <domid>                                      │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Xen Hypervisor                                          │   │
│  │  Provides: vCPU │ memory isolation │ netback/blkback     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU │ RAM │ Disk (for Dom0 only) │ NIC                        │
└─────────────────────────────────────────────────────────────────┘

MirageOS UNIKERNEL BUILD PROCESS:
┌─────────────────────────────────────────────────────────────────┐
│  Developer workstation                                          │
│  OCaml application code → mirage configure --target xen        │
│  → mirage build → unikernel.xen (bootable image)               │
│  → deploy to Xen Dom0 → xl create                              │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- MirageOS compiles OCaml application code + only needed OS components into a single bootable image
- No shell, no package manager, no unnecessary drivers — minimal attack surface
- The unikernel boots as a Xen DomU guest — very fast startup (milliseconds)
- Each unikernel only includes what it needs: a web server unikernel has TCP stack + HTTP but no filesystem

## Use Case
High-security application deployment, DNS servers, VPN endpoints, and research environments where minimal attack surface is paramount.

## Pros
- Extremely minimal attack surface
- Very fast boot times (milliseconds)
- Small image sizes (1-5MB)
- Each app isolated in its own VM

## Cons
- OCaml-based — very limited language ecosystem
- Requires recompilation for any change (no dynamic updates)
- Extremely niche expertise required
- Not suitable for general applications

---

# DEPLOYMENT 67 — Bare Metal → KVM → IncludeOS Unikernels

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              UNIKERNEL SERVICE                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  C++ Application compiled with IncludeOS                 │   │
│  │  → single bootable image (includes OS + app)             │   │
│  │  → runs as KVM guest                                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU + KVM                                         │
│  qemu-system-x86_64 -enable-kvm -kernel includeos.img          │
│  virtio-net │ virtio-blk │ minimal QEMU config                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (host)                          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
High-performance network functions and security-sensitive services requiring C++ ecosystem. NFV research and specialized cloud services.

---

# DEPLOYMENT 68 — Bare Metal → KVM → OSv Unikernels

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              JVM / POSIX APPLICATION                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Java / Scala / Clojure / C / Ruby application           │   │
│  │  runs on OSv kernel (POSIX compatibility layer)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  OSv replaces OS for single application
┌────────────────────────────▼────────────────────────────────────┐
│              OSv UNIKERNEL (bootable VM image)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Application (Java JAR / native binary)                  │   │
│  │  JVM or language runtime (embedded)                      │   │
│  │  OSv kernel (POSIX subset implementation)                │   │
│  │  virtio-net │ virtio-blk │ minimal device support        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU + KVM                                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
└─────────────────────────────────────────────────────────────────┘
```

## Use Case
Running JVM applications (Spring Boot, etc.) as unikernels for cloud workloads — eliminates full OS overhead for JVM services.

---

# DEPLOYMENT 69 — Bare Metal → KVM / Xen → Unikraft Unikernels

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MULTI-LANGUAGE UNIKERNEL WORKLOADS                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Python app unikernel  │  Go app unikernel  │  C app     │   │
│  │  (Flask server)        │  (gRPC service)    │  unikernel │   │
│  │  Boot time: <1ms       │  Boot time: <1ms   │  <1ms boot │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              UNIKRAFT UNIKERNEL IMAGE                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Application + selected Unikraft micro-libraries          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Only required components (library OS approach):   │  │   │
│  │  │  uknetdev (networking) │ ukblkdev (storage)        │  │   │
│  │  │  uksched (scheduler)   │ ukmalloc (memory)         │  │   │
│  │  │  libc port │ language runtime port │ app           │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Compiled for target platform: kvm / xen / linuxu        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  boots as KVM or Xen guest
┌────────────────────────────▼────────────────────────────────────┐
│              KVM (qemu-system-x86_64 -enable-kvm)               │
│              OR Xen (xl create)                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (host) + KVM module                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Intel VT-x or AMD-V │ RAM │ NIC                               │
└─────────────────────────────────────────────────────────────────┘

UNIKRAFT BUILD PROCESS:
┌─────────────────────────────────────────────────────────────────┐
│  Developer workstation                                          │
│  kraft build --plat qemu --arch x86_64                         │
│  → selects required micro-libraries                            │
│  → compiles unikernel image                                    │
│  kraft run                                                     │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- Unikraft is a modular library OS — applications select only required OS components at compile time
- This produces a minimal unikernel image that runs as a KVM/Xen guest
- Boot times can be under 1ms for simple services
- Multiple language ports available (Python, Go, Rust, C, C++, Java)
- Linux Foundation project with active university research backing

## Use Case
Edge computing and FaaS research. Organizations exploring ultra-fast VM startup for function-per-request workloads. Research at Cambridge, Manchester universities.

## Pros
- Multiple language support (Python, Go, C, Rust)
- Under 1ms boot times possible
- Library OS approach — modular and maintainable
- Active Linux Foundation project

## Cons
- Research-grade for production
- Requires application porting effort
- Limited POSIX compatibility
- Small production deployment base

---

# DEPLOYMENT 70 — Bare Metal → OpenStack + Kubernetes → VNFs (OPNFV)

## Box Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              NETWORK SERVICES (end-user perspective)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Mobile subscribers │ Internet customers │ Enterprise VPN │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              NFV SERVICE LAYER                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MANO (Management and Orchestration)                     │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  NFV Orchestrator (NFVO) — OSM / ONAP              │  │   │
│  │  │  VNF Manager (VNFM) — per-VNF lifecycle            │  │   │
│  │  │  Virtualized Infrastructure Manager (VIM)          │  │   │
│  │  │  → OpenStack (for VM-based VNFs)                   │  │   │
│  │  │  → Kubernetes (for container-based CNFs)            │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  NETWORK FUNCTIONS (VNFs / CNFs)                         │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  VM-based VNFs (on OpenStack):                     │  │   │
│  │  │  vFirewall │ vLoadBalancer │ vRouter │ vIMS        │  │   │
│  │  │  vEPC (mobile core) │ vBNG │ vCPE                  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  CNFs (on Kubernetes):                             │  │   │
│  │  │  free5GC (5G core) │ Open5GS │ SD-Core             │  │   │
│  │  │  Contiv-VPP (fast data path)                       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              NFVI (Network Functions Virtualization Infra)      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenStack (VIM for VNFs)                                │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Nova + KVM (VNF VM compute)                       │  │   │
│  │  │  Neutron + OVS/DPDK (high-perf NFV networking)     │  │   │
│  │  │  SR-IOV (NIC passthrough for line-rate performance) │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes (VIM for CNFs)                               │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  DPDK-enabled pods │ SR-IOV device plugin          │  │   │
│  │  │  Multus CNI (multiple NICs per pod)                │  │   │
│  │  │  CPU pinning (NUMA-aware scheduling)               │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              PERFORMANCE-OPTIMIZED HOST OS                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  DPDK (Data Plane Development Kit — bypass kernel net)   │   │
│  │  HugePages (2MB/1GB pages for DPDK memory)               │  │   │
│  │  CPU isolation (isolcpus= for DPDK and VNF pinning)      │   │
│  │  NUMA topology awareness                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (COTS hardware)                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Compute nodes: Intel Xeon (DPDK-optimized) │ RAM 512GB+ │   │
│  │  NIC: Intel X710/XXV710 (SR-IOV capable) 25GbE+          │   │
│  │  BIOS: HugePages enabled │ VT-x │ VT-d (IOMMU)          │   │
│  │  (VT-d required for SR-IOV NIC passthrough)              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## How It Works
- OPNFV is a reference architecture for telecom NFV — it defines how OpenStack + Kubernetes work together for network functions
- OpenStack provides VMs for traditional VNFs (vFirewall, vRouter, vEPC)
- Kubernetes provides pods for cloud-native CNFs (free5GC 5G core)
- DPDK bypasses the Linux kernel network stack for line-rate packet processing
- SR-IOV passes physical NIC virtual functions directly to VMs/pods
- HugePages reduce TLB pressure for DPDK memory management
- MANO layer (OSM/ONAP) orchestrates VNF lifecycle across both OpenStack and K8s

## Use Case
Mobile operators, ISPs, and telcos replacing proprietary network hardware with software running on commodity servers. 4G/5G core networks, SD-WAN, virtual firewalls, and load balancers.

## Pros
- Eliminates expensive proprietary network hardware
- Software-defined network functions
- Carrier-grade HA and performance with DPDK + SR-IOV
- Open source reference architecture

## Cons
- Extremely complex (telco expertise + OpenStack + K8s + DPDK)
- Requires DPDK-capable hardware and careful BIOS tuning
- Long certification cycles
- Very high operational knowledge requirement

---

# QUICK REFERENCE TABLE: Deployments 31–70

| # | Stack | Box Layers (Bottom→Top) | Best For | Complexity |
|---|-------|------------------------|---------|-----------|
| 31 | Docker Swarm | Bare Metal → Linux → Docker → Swarm | Small team containers | Low-Med |
| 32 | Nomad | Bare Metal → Linux → Nomad → Containers+VMs | HashiCorp multi-workload | Medium |
| 33 | Apptainer (HPC) | Bare Metal → Linux → SLURM → Apptainer | HPC scientific computing | Medium |
| 34 | KVM → VMs → K8s | Bare Metal → KVM → VMs → Kubernetes | Enterprise K8s clusters | High |
| 35 | KVM → VMs → Docker | Bare Metal → KVM → VMs → Docker/LXC | Multi-tenant hosting | High |
| 36 | LXD → K8s nodes | Bare Metal → LXD cluster → K8s in containers | High-density K8s labs | High |
| 37 | LXD unified | Bare Metal → LXD → Containers + VMs | SMB unified workloads | Med-High |
| 38 | K8s → KubeVirt | Bare Metal → K8s → KubeVirt → VMs | Unified VM+container | High |
| 39 | K8s → Kata | Bare Metal → K8s → Kata → MicroVM per pod | Multi-tenant K8s security | High |
| 40 | K8s → Firecracker | Bare Metal → K8s → Kata-FC → Firecracker VMs | K8s serverless isolation | High |
| 41 | Harvester | Bare Metal → RKE2 → KubeVirt + Longhorn | VMware migration (OSS) | High |
| 42 | OpenShift + KubeVirt | Bare Metal → RHCOS → OCP → KubeVirt VMs | Red Hat enterprise | Very High |
| 43 | Talos → K8s | Bare Metal → Talos → K8s → KubeVirt | Security-first K8s | High |
| 44 | k3s → KubeVirt | Bare Metal (edge) → k3s → KubeVirt | Edge VM+container | Med-High |
| 45 | MicroShift | Bare Metal (edge) → RHEL → MicroShift | RH edge devices | Medium |
| 46 | OpenStack → KVM | Bare Metal → OpenStack → Nova → KVM VMs | Private cloud IaaS | Very High |
| 47 | OpenStack Ironic | Bare Metal → Ironic → provisioned nodes | Bare metal cloud API | High |
| 48 | OpenStack Magnum | Bare Metal → OpenStack → Magnum → K8s | K8s-as-a-Service | Very High |
| 49 | OpenStack → VMs → K8s | Bare Metal → OpenStack → VMs → K8s | Legacy enterprise cloud | Very High |
| 50 | K8s → OpenStack pods | Bare Metal → K8s → OpenStack pods → KVM | GitOps OpenStack | Extreme |
| 51 | MAAS → OpenStack/K8s | Bare Metal → MAAS → OS/K8s/KVM | Ubuntu bare metal provisioning | High |
| 52 | K8s → Metal3 | Bare Metal → K8s → Metal3/Ironic → nodes | GitOps bare metal K8s | High |
| 53 | Ironic standalone | Bare Metal → Ironic → K8s nodes | Lightweight bare metal prov | High |
| 54 | Warewulf HPC | Bare Metal → Warewulf → diskless cluster | HPC supercomputing | Med |
| 55 | CloudStack | Bare Metal → CloudStack → KVM/Xen/VMware | Hosting providers | High |
| 56 | OpenNebula | Bare Metal → OpenNebula → KVM/LXD/VMware | Research/EU cloud | Med-High |
| 57 | Eucalyptus | Bare Metal → Eucalyptus → KVM/Xen | Historical/deprecated | N/A |
| 58 | Ceph + OpenStack | Bare Metal → Ceph + OpenStack → KVM VMs | Large private cloud | Very High |
| 59 | Ceph + K8s + KubeVirt | Bare Metal → Rook-Ceph → K8s → KubeVirt | Cloud-native HCI | Very High |
| 60 | GlusterFS + KVM | Bare Metal → GlusterFS + KVM → VMs | SMB shared storage VMs | Medium |
| 61 | LINSTOR + KVM | Bare Metal → DRBD/LINSTOR → KVM VMs | Database HA, zero RPO | High |
| 62 | Firecracker standalone | Bare Metal → Linux → Firecracker MicroVMs | Serverless FaaS at scale | High |
| 63 | Cloud Hypervisor | Bare Metal → Linux → Cloud Hypervisor VMs | Cloud MicroVM research | High |
| 64 | K8s + OpenFaaS + FC | Bare Metal → K8s → OpenFaaS → Firecracker | Self-hosted serverless | Very High |
| 65 | K8s + Knative + KubeVirt | Bare Metal → K8s → Knative → KubeVirt VMs | Serverless VM (experimental) | Extreme |
| 66 | Xen + MirageOS | Bare Metal → Xen → MirageOS unikernels | Security research | Very High |
| 67 | KVM + IncludeOS | Bare Metal → KVM → IncludeOS unikernels | NFV research, C++ apps | High |
| 68 | KVM + OSv | Bare Metal → KVM → OSv unikernels | JVM unikernel workloads | High |
| 69 | KVM/Xen + Unikraft | Bare Metal → KVM/Xen → Unikraft unikernels | Edge FaaS research | High |
| 70 | OPNFV | Bare Metal → OpenStack+K8s → DPDK → VNFs | Telco NFV, 5G core | Extreme |

---
*Part 2 of 4 — Deployments 31–70 with complete box architectures*
*Part 1 covers Deployments 1–30 | Part 3 will cover 71–90 | Part 4 will cover 91–110*
