[← Back to Index](../index.md)

# DEPLOYMENT 69 — Bare Metal → KVM / Xen → Unikraft Unikernels

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│              Unikraft Unikernels               │
├──────────────────────────────────────────────────┤
│                   KVM / Xen                    │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


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