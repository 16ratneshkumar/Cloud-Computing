[← Back to Index](../index.md)

# DEPLOYMENT 55 — Bare Metal → Apache CloudStack → KVM/Xen/VMware → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                 KVM/Xen/VMware                 │
├──────────────────────────────────────────────────┤
│               Apache CloudStack                │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT / ADMIN INTERFACES                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CloudStack UI (web)  │  cloudmonkey CLI                 │   │
│  │  AWS EC2-compatible API (CloudStack implements EC2 API)  │   │
│  │  Terraform CloudStack provider                           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  HTTP/HTTPS API
┌────────────────────────────▼────────────────────────────────────┐
│              APACHE CLOUDSTACK MANAGEMENT SERVER                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CloudStack Management Server (Java-based)               │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Zone / Pod / Cluster / Host hierarchy             │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  Zone (data center)                          │  │  │   │
│  │  │  │  └── Pod (server rack group)                 │  │  │   │
│  │  │  │       └── Cluster (same hypervisor type)     │  │  │   │
│  │  │  │            └── Hosts (hypervisor nodes)      │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  │  VM Orchestration │ Network Mgmt │ Storage Mgmt    │  │   │
│  │  │  Account / Domain hierarchy (multi-tenancy)        │  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  MySQL (management DB) │ NFS / Ceph (secondary storage)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  CloudStack agent on each host
┌────────────────────────────▼────────────────────────────────────┐
│              HYPERVISOR HOSTS                                   │
│  ┌────────────────────┐  ┌────────────────┐  ┌──────────────┐  │
│  │  KVM Host          │  │  XCP-ng (Xen)  │  │  VMware ESXi │  │
│  │  CloudStack agent  │  │  CloudStack    │  │  Host        │  │
│  │  libvirt           │  │  agent         │  │              │  │
│  │  ┌──────────────┐  │  │  ┌──────────┐  │  │  ┌────────┐ │  │
│  │  │ Tenant VMs   │  │  │  │ VMs      │  │  │  │ VMs    │ │  │
│  │  └──────────────┘  │  │  └──────────┘  │  │  └────────┘ │  │
│  └────────────────────┘  └────────────────┘  └──────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Hypervisor hosts │ NFS/Ceph storage │ management network      │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- CloudStack Management Server is a single Java application (simpler than OpenStack's ~30 services)
- Zone/Pod/Cluster/Host hierarchy maps to data center physical structure
- CloudStack agent on each host receives commands from management server
- Supports KVM, XCP-ng/Xen, and VMware ESXi as hypervisors
- AWS EC2 API compatibility allows AWS tooling (CLI/SDK) to work against CloudStack

## Use Case
Hosting providers building public/private clouds wanting AWS API compatibility. Simpler to operate than OpenStack. Popular in Asia-Pacific cloud market.

## Pros
- Simpler deployment than OpenStack (single Java process)
- Supports multiple hypervisors from one management plane
- AWS EC2-compatible API
- Strong multi-tenancy with account/domain hierarchy

## Cons
- Smaller community than OpenStack/Kubernetes
- Less modular architecture
- Limited modern cloud-native features (no K8s integration built-in)
- Fewer public cloud providers using it