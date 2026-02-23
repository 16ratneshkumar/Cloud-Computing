[← Back to Index](../index.md)

# DEPLOYMENT 99 — Bare Metal → Nomad + Firecracker → MicroVM Workloads

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│               MicroVM Workloads                │
├──────────────────────────────────────────────────┤
│              Nomad + Firecracker               │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MIXED WORKLOADS (Nomad-scheduled)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container jobs (docker driver)                          │   │
│  │  Firecracker MicroVM jobs (firecracker-task-driver)      │   │
│  │  Binary jobs (exec driver)                               │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  job placement
┌────────────────────────────▼────────────────────────────────────┐
│              NOMAD (orchestration)                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Nomad Servers (Raft consensus)                          │   │
│  │  Nomad Clients (worker nodes)                            │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Task Drivers:                                     │  │   │
│  │  │  docker → containerd (standard containers)         │  │   │
│  │  │  firecracker-task-driver → Firecracker MicroVMs    │  │   │
│  │  │  exec → raw binaries (host process)                │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Consul (service discovery + health checks)              │   │
│  │  Vault (secrets injection into tasks)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              FIRECRACKER (per MicroVM task)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  jailer + firecracker binary per MicroVM task            │   │
│  │  virtio-net (TAP) │ virtio-blk │ minimal Linux kernel    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/kvm
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL                                 │
│  kvm.ko │ TUN/TAP │ cgroups │ seccomp (jailer)                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V) │ RAM │ NVMe │ NIC                          │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
HashiCorp-ecosystem teams wanting MicroVM isolation without Kubernetes complexity — Nomad's simplicity + Firecracker's security for mixed workload platforms.

## Pros
- Nomad's simplicity vs K8s complexity
- Firecracker's strong isolation for security-sensitive tasks
- Docker + MicroVM + binary workloads from one scheduler
- Consul + Vault integration

## Cons
- Experimental Firecracker driver for Nomad (community, not official)
- Much smaller ecosystem than K8s + Kata/Firecracker
- Limited orchestration features vs K8s