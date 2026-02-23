[← Back to Index](../index.md)

# DEPLOYMENT 50 — Bare Metal → Kubernetes → OpenStack Control Plane (Containerized)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│    OpenStack Control Plane (Containerized)     │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT CLIENTS                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  openstack CLI  │  Horizon  │  kubectl  │  Helm / ArgoCD │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (base layer for OpenStack)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OPENSTACK CONTROL PLANE PODS (via OpenStack-Helm / OSO) │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │  keystone    │  │  nova        │  │  neutron     │   │   │
│  │  │  pod(s)      │  │  pod(s)      │  │  pod(s)      │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │  cinder      │  │  glance      │  │  horizon     │   │   │
│  │  │  pod(s)      │  │  pod(s)      │  │  pod(s)      │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Infrastructure pods:                            │    │   │
│  │  │  MariaDB (Galera)  │  RabbitMQ  │  Memcached    │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  │  K8s manages: restarts, scaling, upgrades of OpenStack   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  nova-compute on PHYSICAL compute nodes
┌────────────────────────────▼────────────────────────────────────┐
│              BARE METAL COMPUTE NODES                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  nova-compute agent (runs directly, not in K8s)          │   │
│  │  libvirt + KVM → launches actual tenant VMs              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │  Tenant VM A │  │  Tenant VM B │  │  Tenant VM C │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  │  OVN agents (neutron networking for VMs)                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  K8s CONTROL NODES (run OpenStack pods):                        │
│  CPU 32c │ RAM 256GB │ NVMe │ 25GbE NIC                        │
│                                                                  │
│  COMPUTE NODES (run tenant KVM VMs):                            │
│  CPU 64c/128t │ RAM 512GB+ │ NVMe │ 25GbE NIC                  │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Kubernetes cluster hosts all OpenStack control plane services as pods (via OpenStack-Helm charts or OpenStack Operator)
- K8s manages OpenStack pods — if nova-api crashes, K8s restarts it automatically
- MariaDB, RabbitMQ, Memcached also run as pods with K8s StatefulSets
- nova-compute still runs directly on bare metal compute nodes (not in K8s) for direct KVM access
- OpenStack API calls flow: tenant → OpenStack pods (in K8s) → nova-compute on bare metal → KVM VMs

## Use Case
Advanced telco and cloud providers wanting GitOps-managed OpenStack with K8s-native self-healing control plane. AT&T, Verizon-scale deployments.

## Pros
- OpenStack upgrades become Helm chart upgrades
- K8s self-heals OpenStack control plane pods
- GitOps-friendly (ArgoCD manages OpenStack)
- Better control plane resource utilization

## Cons
- Extreme complexity (K8s + OpenStack expertise required)
- Debugging spans multiple abstraction layers
- Very limited public documentation and case studies
- Long initial deployment timeline