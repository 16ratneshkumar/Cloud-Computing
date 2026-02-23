[← Back to Index](../index.md)

# DEPLOYMENT 77 — Bare Metal → OpenStack → VMs → OpenStack (Nested)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│               OpenStack (Nested)               │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                   OpenStack                    │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              L2 TENANT VMs (nested cloud)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Tenant L2 VMs (launched via inner OpenStack)            │   │
│  │  These are VMs running inside VMs inside OpenStack       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              INNER OPENSTACK (L1 — running inside L0 VMs)       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Inner OpenStack control plane (Nova, Neutron, Keystone) │   │
│  │  Runs inside L0 Nova VMs (nested KVM enabled)            │   │
│  │  Inner Nova → libvirt → nested KVM → L2 VMs             │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OUTER OPENSTACK L0 (bare metal cloud)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  L0 Nova (provisions L1 OpenStack infra VMs)             │   │
│  │  L0 Neutron (networking for L1 VMs)                      │   │
│  │  L0 VMs: inner-controller, inner-compute × N             │   │
│  │  All with nested KVM enabled (cpu_mode=host-passthrough) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              L0 KVM COMPUTE NODES + LINUX KERNEL                │
│  kvm-intel.ko nested=1 │ libvirt cpu_mode=host-passthrough       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (large — nested overhead 40-70%) │ RAM (massive pool)     │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
OpenStack CI testing (testing OpenStack upgrades using nested OpenStack), development/lab environments, and cloud providers offering "private cloud in a cloud."

## Pros
- Test OpenStack without physical hardware
- Isolated OpenStack environments per project/team
- Useful for telco labs

## Cons
- Extreme overhead (three layers deep)
- Very resource intensive
- Slow I/O and networking at L2
- Production use extremely rare