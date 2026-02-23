[← Back to Index](../index.md)

# DEPLOYMENT 66 — Bare Metal → Xen → MirageOS Unikernels

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│              MirageOS Unikernels               │
├──────────────────────────────────────────────────┤
│                      Xen                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


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