[← Back to Index](../index.md)

# DEPLOYMENT 5 — Bare Metal → OpenBSD → VMM

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMM                       │
├──────────────────────────────────────────────────┤
│                    OpenBSD                     │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                    GUEST VIRTUAL MACHINES                       │
│  ┌──────────────────────────┐  ┌───────────────────────────┐   │
│  │   Linux Guest VM         │  │   OpenBSD Guest VM        │   │
│  │   (Ubuntu / Debian)      │  │   (OpenBSD 7.x)           │   │
│  │   virtio-net / virtio-blk│  │   virtio drivers          │   │
│  │   vio0 (network if)      │  │   vio0 (network if)       │   │
│  └──────────────────────────┘  └───────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  virtio I/O
┌────────────────────────────▼────────────────────────────────────┐
│              OpenBSD VMM (vmm(4) + vmd(8))                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  vmm(4) — kernel VMM driver (Intel VT-x / AMD-V)          │ │
│  │  vmd(8) — userspace VMM management daemon                 │ │
│  │  vmctl  — CLI to create/start/stop VMs                    │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OpenBSD HOST OS                                     │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │  pf (firewall)  │  │  pledge(2)   │  │  unveil(2)         │ │
│  │  (NAT/filter for│  │  (syscall    │  │  (filesystem       │ │
│  │   VM traffic)   │  │   whitelist) │  │   visibility)      │ │
│  └─────────────────┘  └──────────────┘  └────────────────────┘ │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vether(4) / bridge(4) — VM virtual networking           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                 OpenBSD KERNEL                                   │
│  vmm(4) driver │ W^X enforcement │ ASLR │ Stack Canaries        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   Intel VT-x / AMD-V CPU │ RAM │ SSD/HDD │ NIC (em0/bge0)      │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- vmm(4) is the kernel VMM driver using Intel VT-x or AMD-V
- vmd(8) is the userspace management daemon that vmd launches VMs and proxies I/O
- VMs use virtio drivers for network and disk
- pf handles NAT and filtering for VM traffic
- OpenBSD's pledge/unveil further restrict vmd's capabilities

## Use Case
Running Linux services (web servers, databases) behind OpenBSD's security hardening and pf firewall. Security researchers and high-security network appliances.

## Pros
- OpenBSD's world-class security applies to hypervisor
- Minimal, auditable codebase
- pf integration for strict VM network control
- Good for security-conscious labs

## Cons
- No Windows guest support
- No live migration
- Limited guest device support
- Small community — limited documentation
- Performance not competitive with KVM