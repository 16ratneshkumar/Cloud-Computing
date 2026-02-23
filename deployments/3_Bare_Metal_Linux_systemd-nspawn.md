[← Back to Index](../index.md)

# DEPLOYMENT 3 — Bare Metal → Linux → systemd-nspawn

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                 systemd-nspawn                 │
├──────────────────────────────────────────────────┤
│                     Linux                      │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTAINERIZED SERVICES                       │
│  ┌──────────────────┐    ┌───────────────────────────────────┐  │
│  │ Service A        │    │ Service B                         │  │
│  │ (own rootfs dir) │    │ (own rootfs dir)                  │  │
│  │ /var/lib/machines│    │ /var/lib/machines/serviceB        │  │
│  │ /serviceA        │    │                                   │  │
│  └──────────────────┘    └───────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                 systemd-nspawn + systemd-machined                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  systemd-nspawn  (chroot-like container runtime)         │   │
│  │  machinectl      (list/start/stop/login to containers)   │   │
│  │  systemd-machined(machine registry daemon)               │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │             Networking                                   │   │
│  │  systemd-networkd │ veth pairs │ host0 interface         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      systemd (PID 1)                            │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │  journald  │  │  networkd    │  │  resolved              │  │
│  └────────────┘  └──────────────┘  └────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                      LINUX KERNEL                               │
│              cgroups │ namespaces │ seccomp                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│            CPU │ RAM │ Storage │ NIC                           │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Often relies on Linux bridges (`lxcbr0`) and `veth` pairs.
- **Storage:** Local directory or CoW filesystems (ZFS/Btrfs).
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- systemd-nspawn creates lightweight containers using Linux namespaces — essentially a chroot with namespace isolation
- systemd-machined provides a D-Bus API for container lifecycle
- machinectl is the CLI interface
- No external daemon needed — built into every modern Linux

## Use Case
Minimal service isolation, OS image testing, distribution package building, and lightweight dev environments where Docker/LXD is too heavy.

## Pros
- Zero additional software — built into systemd
- Very lightweight
- Good for testing OS images and chroots
- Integrates with journald for container logging

## Cons
- Very limited networking features
- No clustering or live migration
- Not suitable for production multi-tenant workloads
- No image registry or management plane
- Network configuration manual