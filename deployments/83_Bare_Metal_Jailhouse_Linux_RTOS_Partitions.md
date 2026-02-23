[← Back to Index](../index.md)

# DEPLOYMENT 83 — Bare Metal → Jailhouse → Linux + RTOS Partitions

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│            Linux + RTOS Partitions             │
├──────────────────────────────────────────────────┤
│                   Jailhouse                    │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              PARTITIONED WORKLOADS                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Cell 0 (Root Cell)          Cell 1 (Inmate)             │   │
│  │  ┌──────────────────────┐    ┌──────────────────────┐    │   │
│  │  │  Linux (full OS)     │    │  RTOS (bare metal C)  │    │   │
│  │  │  Runs normal Linux   │    │  Hard RT task control  │    │   │
│  │  │  workloads           │    │  Industrial PLC logic  │    │   │
│  │  │  Manages Jailhouse   │    │  Motor controller      │    │   │
│  │  │  (jailhouse CLI)     │    │  No OS overhead        │    │   │
│  │  └──────────────────────┘    └──────────────────────┘    │   │
│  │  Partitioned CPU cores:                                  │   │
│  │  Cell 0: cores 0-5  │  Cell 1: cores 6-7 (dedicated RT) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              JAILHOUSE HYPERVISOR                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Jailhouse (loaded as Linux kernel module)               │   │
│  │  Type-1 static partitioning hypervisor                  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Static memory partitioning (no dynamic allocation) │  │   │
│  │  │  CPU core assignment (per-cell dedicated cores)     │  │   │
│  │  │  Device ownership (per-cell device assignment)      │  │   │
│  │  │  No scheduling — cells run simultaneously           │  │   │
│  │  │  Comm regions (shared memory between cells)         │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Very small hypervisor: ~12,000 lines of C code          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  loads over running Linux kernel
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (pre-Jailhouse load)                  │
│  Linux boots normally → jailhouse enable → Jailhouse takes over │
│  Linux demoted to Cell 0 root cell                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  x86 multi-core / ARM multi-core (tested on i.MX8, Jetson)     │
│  Intel VT-x / VT-d required for x86                            │
│  RAM │ Peripherals divided between cells                        │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Linux boots normally, then `jailhouse enable cell0.cfg` loads the Jailhouse hypervisor
- Jailhouse takes over hardware and demotes Linux to the "root cell"
- Inmate cells are created: `jailhouse cell create rtos-cell.cfg`
- Cells get dedicated CPU cores, memory regions, and devices — no sharing
- RTOS inmate runs on dedicated cores with zero interference from Linux

## Use Case
Industrial automation, robotics, and automotive — running Linux (for general compute + networking) and a hard RT RTOS (for motor control, sensor reading) on the same SoC.

## Pros
- Hard real-time in inmate cell with zero Linux interference
- Small hypervisor code (auditable)
- Existing Linux system can be "Jailhoused" without full redesign
- Good Siemens/industry backing

## Cons
- Static partitioning — cannot change partition config at runtime
- Requires hardware VT-x/VT-d
- Limited to shared-memory IPC between cells
- Complex board support configuration files