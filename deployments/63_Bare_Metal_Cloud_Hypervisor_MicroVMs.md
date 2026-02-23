[← Back to Index](../index.md)

# DEPLOYMENT 63 — Bare Metal → Cloud Hypervisor → MicroVMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                    MicroVMs                    │
├──────────────────────────────────────────────────┤
│                Cloud Hypervisor                │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

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

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


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