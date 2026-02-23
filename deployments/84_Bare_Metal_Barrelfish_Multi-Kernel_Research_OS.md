[← Back to Index](../index.md)

# DEPLOYMENT 84 — Bare Metal → Barrelfish → Multi-Kernel Research OS

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│            Multi-Kernel Research OS            │
├──────────────────────────────────────────────────┤
│                   Barrelfish                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              RESEARCH APPLICATIONS                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Research workloads (custom Barrelfish apps)             │   │
│  │  Network stack experiments │ OS research │ HPC protos    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              BARRELFISH (Multi-Kernel OS)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  One CPU Core = One CPU Driver (like a separate kernel)  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Core 0: CPU driver 0 (manages core 0 resources)   │  │   │
│  │  │  Core 1: CPU driver 1 (manages core 1 resources)   │  │   │
│  │  │  Core 2: CPU driver 2 ...                          │  │   │
│  │  │  Core N: CPU driver N                              │  │   │
│  │  │  → Each core runs its own kernel-like driver       │  │   │
│  │  │  → No shared memory between cores (message passing)│  │   │
│  │  │  → System Knowledge Base (SKB) — distributed fact  │  │   │
│  │  │    database for system configuration               │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Inter-core communication: URPC (userspace RPC, lockless)│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Many-core x86/ARM (designed for 1000+ cores)                   │
│  NUMA topology │ Large RAM │ NIC                               │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Academic OS research, particularly for heterogeneous many-core systems. ETH Zurich + Microsoft Research project.

## Pros / Cons
- Research-only — not for production
- Groundbreaking architecture ideas (multi-kernel model)