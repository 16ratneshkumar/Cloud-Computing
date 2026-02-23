[← Back to Index](../index.md)

# DEPLOYMENT 15 — Bare Metal → XCP-ng → Xen → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                      Xen                       │
├──────────────────────────────────────────────────┤
│                     XCP-ng                     │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              XEN ORCHESTRA (Web Management)                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Xen Orchestra (XO) — Node.js web application            │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │   │
│  │  │  VM Mgmt    │  │  Backup &    │  │  Cloud-init     │ │   │
│  │  │  (create,   │  │  Replication │  │  integration    │ │   │
│  │  │  migrate,   │  │  (XO Lite or │  │                 │ │   │
│  │  │  snapshots) │  │   XO Pro)    │  │                 │ │   │
│  │  └─────────────┘  └──────────────┘  └─────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  XAPI protocol
┌────────────────────────────▼────────────────────────────────────┐
│              XCP-ng HOST                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Dom0 (CentOS-based control domain)                       │   │
│  │  ┌────────────────┐  ┌──────────────────────────────┐    │   │
│  │  │  XAPI (toolstack│  │ xcp-networkd (network mgmt) │    │   │
│  │  │  — XAPI daemon) │  └──────────────────────────────┘    │   │
│  │  └────────────────┘                                       │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │             GUEST VMs (DomU)                       │   │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌─────────┐  │   │   │
│  │  │  │ Windows VM   │  │ Linux VM     │  │ BSD VM  │  │   │   │
│  │  │  │ HVM + tools  │  │ PV / HVM     │  │ HVM     │  │   │   │
│  │  │  └──────────────┘  └──────────────┘  └─────────┘  │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  │  ┌────────────────────────────────────────────────────┐   │   │
│  │  │  Storage Repositories (SR)                         │   │   │
│  │  │  LVM SR │ NFS SR │ iSCSI SR │ ZFS SR (community)  │   │   │
│  │  └────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  XEN HYPERVISOR                                  │
│  vCPU Scheduler (credit2) │ Memory Management │ Event Channels  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│   CPU (VT-x/AMD-V) │ RAM │ Local SSD / SAN / NAS │ NIC          │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Open-source Citrix XenServer alternative. Community-driven enterprise Xen platform managed by Xen Orchestra. Popular VMware/Citrix alternative in Europe.

## Pros
- Free open-source Xen platform
- Xen Orchestra provides excellent management UI
- Compatible with Citrix XenServer API and tools
- Good Windows VM support

## Cons
- Xen architecture adds Dom0 complexity
- Smaller ecosystem than KVM-based platforms
- Xen declining vs KVM long-term