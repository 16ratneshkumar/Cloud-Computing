[← Back to Index](../index.md)

# DEPLOYMENT 32 — Bare Metal → Nomad → Containers + QEMU VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│             Containers + QEMU VMs              │
├──────────────────────────────────────────────────┤
│                     Nomad                      │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


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