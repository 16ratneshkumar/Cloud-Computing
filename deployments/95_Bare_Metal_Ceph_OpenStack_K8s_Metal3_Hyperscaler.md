[← Back to Index](../index.md)

# DEPLOYMENT 95 — Bare Metal → Ceph + OpenStack + K8s + Metal3 (Hyperscaler)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│ Ceph + OpenStack + K8s + Metal3 (Hyperscaler)  │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MULTI-TIER WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Tenant VMs (OpenStack tenants via Horizon/Nova)         │   │
│  │  Bare Metal nodes (OpenStack Ironic tenants)             │   │
│  │  Kubernetes clusters (Magnum / external K8s on Ironic)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LAYER 3: OPENSTACK (IaaS control plane)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Nova (KVM VMs) │ Ironic (bare metal) │ Neutron (SDN)    │   │
│  │  Cinder (→ Ceph RBD) │ Glance (→ Ceph RBD) │ Swift (RGW) │   │
│  │  Magnum (K8s-as-a-Service) │ Heat (orchestration)        │   │
│  │  Keystone │ Barbican │ Designate │ Octavia               │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LAYER 2: MANAGEMENT KUBERNETES CLUSTER             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenStack control plane pods (OpenStack-Helm / OSO)     │   │
│  │  Metal3 + Ironic (bare metal provisioning)               │   │
│  │  Cluster API (manages K8s cluster provisioning)          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LAYER 1: CEPH STORAGE CLUSTER                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RBD (block) → Cinder volumes, Nova ephemeral, Glance    │   │
│  │  RGW (object) → Swift API, backup storage                │   │
│  │  CephFS (file) → Manila shared filesystems               │   │
│  │  50+ OSD nodes, petabytes of storage                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LAYER 0: BARE METAL INFRASTRUCTURE                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Control nodes:  CPU 32c │ RAM 256GB │ NVMe │ 25GbE      │   │
│  │  Compute nodes:  CPU 128c│ RAM 1TB+  │ NVMe │ 25GbE×2   │   │
│  │  Storage nodes:  CPU 16c │ RAM 64GB  │ 12×HDD+2×NVMe    │   │
│  │                          │           │ 25GbE×2           │   │
│  │  Network nodes:  CPU 8c  │ RAM 32GB  │ 100GbE spine      │   │
│  │  Managed via: Metal3 / MAAS / Ironic standalone          │   │
│  └──────────────────────────────────────────────────────────┘   │
│  Physical network:                                              │
│  Leaf-spine topology │ BGP EVPN │ 100GbE spine │ 25GbE leaf   │
└─────────────────────────────────────────────────────────────────┘

FULL CALL CHAIN:
kubectl apply BareMetalHost → Metal3/Ironic powers on node → PXE
→ OS deploy (RHEL/Ubuntu) → Ironic provisions → Nova registers
→ Tenant requests VM → Nova schedules → KVM VM → Ceph RBD disk
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- This is the extreme full-stack: Kubernetes manages OpenStack which manages KVM VMs and bare metal, all backed by Ceph
- Metal3 + Ironic handle all physical node provisioning via Redfish/IPMI
- OpenStack control plane pods run in the management K8s cluster (self-healing via K8s)
- All storage (VM disks, images, objects, shared filesystems) routes through Ceph
- Tenant interface: OpenStack Horizon or Nova/Cinder API — they get VMs, bare metal, K8s clusters

## Use Case
Large private cloud operators (national research networks, telcos, universities with 1000+ servers) building AWS/Azure-scale private infrastructure.

## Pros
- Complete private cloud IaaS with every service
- Bare metal, VM, and K8s workloads from one API
- No vendor lock-in (all open source)
- Scales to thousands of nodes

## Cons
- Requires multiple dedicated teams (Ceph, OpenStack, K8s, Networking, HW)
- Years to design, deploy, and stabilize
- Operational complexity is extreme
- Not for organizations under 500 servers