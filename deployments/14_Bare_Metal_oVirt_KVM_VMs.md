[← Back to Index](../index.md)

# DEPLOYMENT 14 — Bare Metal → oVirt → KVM VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                    KVM VMs                     │
├──────────────────────────────────────────────────┤
│                     oVirt                      │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                  MANAGEMENT CLIENTS                             │
│  ┌──────────────────┐  ┌─────────────────┐  ┌───────────────┐  │
│  │  oVirt Web UI    │  │  REST API        │  │  Ansible      │  │
│  │  (Admin/User     │  │  (v4 API)        │  │  oVirt module │  │
│  │   Portal)        │  │                  │  │               │  │
│  └──────────────────┘  └─────────────────┘  └───────────────┘  │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│              oVirt ENGINE (Management Server)                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ovirt-engine (Java EE application — JBoss/WildFly)      │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐ │   │
│  │  │  Scheduler  │  │  Network Mgmt│  │  Storage Mgmt   │ │   │
│  │  │  (VM        │  │  (logical    │  │  (storage        │ │   │
│  │  │   placement)│  │   networks,  │  │   domains: NFS,  │ │   │
│  │  │             │  │   VLANs)     │  │   iSCSI, FC,    │ │   │
│  │  └─────────────┘  └──────────────┘  │   Ceph RBD)     │ │   │
│  │  ┌──────────────────────────────┐   └─────────────────┘ │   │
│  │  │  HA / Migration / Cluster    │                        │   │
│  │  └──────────────────────────────┘                        │   │
│  │  PostgreSQL (config/state DB)                            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────┘
                         │  VDSM agent on each host
┌────────────────────────▼────────────────────────────────────────┐
│              oVirt HOSTS (KVM hypervisor nodes)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VDSM (oVirt host agent) — manages KVM on this host      │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │               KVM Virtual Machines                  │ │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │ │   │
│  │  │  │ VM A         │  │ VM B         │  │ VM C     │  │ │   │
│  │  │  │ Windows      │  │ Linux        │  │ Linux    │  │ │   │
│  │  │  └──────────────┘  └──────────────┘  └──────────┘  │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│              KVM + QEMU + LINUX KERNEL                          │
│   kvm.ko │ libvirt │ TUN/TAP │ Open vSwitch                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                       BARE METAL                                │
│         CPU (VT-x/AMD-V) │ RAM │ Shared Storage │ NIC          │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- oVirt Engine (Java EE app) runs as the central management server with a PostgreSQL database
- VDSM (Virtual Desktop and Server Manager) agent runs on each KVM host and implements oVirt's host API
- Engine communicates with VDSM via XML-RPC to manage VMs on each host
- Storage domains (NFS/iSCSI/FC) are shared across hosts for VM live migration

## Use Case
Open-source alternative to VMware vSphere. Used by organizations wanting enterprise VM management without VMware licensing costs. Upstream of Red Hat Virtualization.

## Pros
- Free, enterprise-grade VM management
- Live migration, HA, storage management
- REST API for automation

## Cons
- Red Hat Virtualization (commercial oVirt) went EOL June 2024
- oVirt community maintenance uncertain
- Complex initial setup
- Proxmox is often chosen over oVirt now