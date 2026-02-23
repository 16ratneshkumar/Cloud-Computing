[← Back to Index](../index.md)

# DEPLOYMENT 4 — Bare Metal → FreeBSD → Jails

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                     Jails                      │
├──────────────────────────────────────────────────┤
│                    FreeBSD                     │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                    JAIL WORKLOADS                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Jail A     │  │   Jail B     │  │   Jail C (VNET)      │  │
│  │ (thin jail)  │  │ (thick jail) │  │ (own network stack)  │  │
│  │ shared base  │  │ full copy    │  │ own routing table    │  │
│  │ /usr/jails/A │  │ /usr/jails/B │  │ own firewall (pf)    │  │
│  │ IP: 10.0.0.1 │  │ IP: 10.0.0.2 │  │ IP: 192.168.10.1    │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│               JAIL MANAGEMENT TOOLS                             │
│  ┌──────────────┐  ┌───────────┐  ┌────────────────────────┐   │
│  │  /etc/jail   │  │  iocage   │  │  ezjail                │   │
│  │  .conf       │  │  (Python  │  │  (shell scripts)       │   │
│  │  (native)    │  │   mgmt)   │  │                        │   │
│  └──────────────┘  └───────────┘  └────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    FREEBSD KERNEL                               │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐   │
│  │  Jail Subsystem  │  │  pf (firewall)   │  │  ZFS        │   │
│  │  (kernel jails)  │  │  (per jail ACL)  │  │  (datasets  │   │
│  │                  │  │                  │  │   per jail) │   │
│  └──────────────────┘  └──────────────────┘  └─────────────┘   │
│  ┌──────────────────┐  ┌──────────────────────────────────┐    │
│  │  VNET (Virtual   │  │  MAC framework (Mandatory Access  │    │
│  │  Network Stack)  │  │  Control)                        │    │
│  └──────────────────┘  └──────────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  CPU │ RAM │ ZFS Pool (mirror/RAIDZ) │ NIC (em0/igb0) │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- FreeBSD kernel's jail subsystem creates isolated environments at the kernel level — process, filesystem, and network isolated
- VNET jails get their own complete network stack (routing table, firewall, interfaces)
- ZFS datasets are assigned per jail for storage isolation
- pf firewall enforces network rules between jails and to the outside
- iocage or ezjail provide management tooling on top of native jail commands

## Use Case
Secure multi-tenant hosting on FreeBSD, web hosting panels (like HestiaCP on BSD), firewall appliances, and security-focused deployments. Pre-dates Linux containers by years.

## Pros
- Very mature and battle-tested (since FreeBSD 4.0, year 2000)
- VNET provides full network stack isolation per jail
- ZFS per-jail dataset isolation is excellent
- Strong security model with MAC framework
- Minimal overhead

## Cons
- FreeBSD only — Linux binaries need Linux compat layer
- Limited Linux container tooling ecosystem
- No native Kubernetes integration
- iocage has had maintenance issues
- Smaller community than Linux containers