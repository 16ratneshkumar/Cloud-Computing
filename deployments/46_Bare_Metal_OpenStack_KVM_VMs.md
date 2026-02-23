[← Back to Index](../index.md)

# DEPLOYMENT 46 — Bare Metal → OpenStack → KVM VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                    KVM VMs                     │
├──────────────────────────────────────────────────┤
│                   OpenStack                    │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT / USER INTERFACES                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Horizon (web dashboard)  │  OpenStack CLI (openstack *)  │   │
│  │  Terraform openstack prov │  Heat (orchestration templates)│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  REST APIs (HTTPS)
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK CORE SERVICES                            │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  KEYSTONE (Identity)                                       │ │
│  │  ┌────────────────────────────────────────────────────┐   │ │
│  │  │  Authentication │ Authorization │ Service Catalog  │   │ │
│  │  │  Tokens (Fernet) │ Projects/Domains/Users          │   │ │
│  │  └────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  NOVA (Compute)                                            │ │
│  │  ┌────────────────────────────────────────────────────┐   │ │
│  │  │  nova-api  │  nova-scheduler  │  nova-conductor     │   │ │
│  │  │  nova-compute (runs on each hypervisor node)        │   │ │
│  │  │  → calls libvirt → launches QEMU/KVM VMs           │   │ │
│  │  └────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  NEUTRON (Networking)                                      │ │
│  │  ┌────────────────────────────────────────────────────┐   │ │
│  │  │  neutron-server │ neutron-agent │ ML2 plugin       │   │ │
│  │  │  OVN / OVS backend                                 │   │ │
│  │  │  Tenant networks │ Floating IPs │ Security Groups  │   │ │
│  │  │  Load Balancer (Octavia) │ VPN (Neutron VPN)       │   │ │
│  │  └────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  CINDER (Block Storage)  │  GLANCE (Image Service)         │ │
│  │  ┌──────────────────┐    │  ┌──────────────────────────┐  │ │
│  │  │  cinder-api       │    │  │  Image registry          │  │ │
│  │  │  cinder-scheduler │    │  │  (qcow2/raw/ISO images)  │  │ │
│  │  │  cinder-volume    │    │  │  Backends: Swift/Ceph/   │  │ │
│  │  │  LVM/Ceph backend │    │  │  local filesystem        │  │ │
│  │  └──────────────────┘    │  └──────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  SWIFT (Object Storage, optional)                          │ │
│  │  MANILA (Shared Filesystem, optional)                      │ │
│  │  BARBICAN (Secrets Management, optional)                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  MESSAGE BUS: RabbitMQ / Oslo.messaging                    │ │
│  │  DATABASE:    MariaDB/MySQL (Galera cluster)               │ │
│  │  CACHE:       Memcached / Redis                            │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────┘
                             │  nova-compute on each hypervisor node
┌────────────────────────────▼────────────────────────────────────┐
│              COMPUTE NODES (hypervisor hosts)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  nova-compute agent                                      │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  libvirt + QEMU/KVM                               │  │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌────────┐  │  │   │
│  │  │  │ Tenant VM A  │  │ Tenant VM B  │  │ VM N   │  │  │   │
│  │  │  │ (Ubuntu inst)│  │ (Win Server) │  │        │  │  │   │
│  │  │  └──────────────┘  └──────────────┘  └────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  neutron-agent (OVN/OVS for VM network plumbing)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (compute nodes)                 │
│  kvm.ko │ kvm-intel/amd.ko │ OVS/OVN │ libvirt                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CONTROLLER NODES (OpenStack services):                        │
│  CPU 16c │ RAM 128GB │ SSD (OS + DB + MQ) │ 10GbE×2            │
│                                                                  │
│  COMPUTE NODES (hypervisor):                                    │
│  CPU 64c/128t (EPYC/Xeon) │ RAM 512GB+ │ NVMe │ 25GbE×2        │
│                                                                  │
│  STORAGE NODES (if Ceph):                                       │
│  CPU 16c │ RAM 64GB │ 12×HDD + 2×NVMe (WAL/DB) │ 25GbE×2      │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Keystone authenticates all API requests and provides a service catalog (endpoints for each service)
- Nova receives VM launch requests, nova-scheduler picks a compute host, nova-compute uses libvirt to start QEMU/KVM
- Neutron creates virtual networks using OVN/OVS, plugging VM vNICs into tenant networks via OVS patch ports
- Cinder provisions block volumes (LVM/Ceph) and attaches them to VMs via iSCSI or virtio-blk
- All services communicate via RabbitMQ message queue; state stored in MariaDB

## Use Case
Private cloud for telecoms, research institutions, large enterprises — replacing AWS/Azure with on-premises IaaS. Tenant self-service VM provisioning with full network isolation.

## Pros
- Full IaaS private cloud — AWS-like self-service
- Multi-tenant with strong isolation
- Enormous feature set (compute, network, storage, identity, object)
- Open source with commercial support (Red Hat, Canonical, SUSE)

## Cons
- Extremely complex to deploy and operate
- High operational overhead (requires dedicated OpenStack team)
- Slow release cycles
- Overkill for small environments