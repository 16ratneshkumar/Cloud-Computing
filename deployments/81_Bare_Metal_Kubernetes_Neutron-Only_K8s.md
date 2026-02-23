[← Back to Index](../index.md)

# DEPLOYMENT 81 — Bare Metal → Kubernetes → Neutron-Only + K8s

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│               Neutron-Only + K8s               │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Pods with Neutron-provided networking                   │   │
│  │  (Kuryr CNI: K8s pods on Neutron subnets)               │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Kuryr CNI driver
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (CNI = Kuryr → Neutron)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kuryr-controller (K8s → Neutron translation)            │   │
│  │  kuryr-cni (pod NIC provisioning via Neutron ports)      │   │
│  │  → K8s Services → Neutron LBaaS (Octavia)               │   │
│  │  → K8s NetworkPolicy → Neutron Security Groups          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK NEUTRON (network only)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  neutron-server │ ML2 plugin │ OVN backend               │   │
│  │  Provides: virtual networks, routers, FIPs, SGs, LBaaS  │   │
│  │  WITHOUT running full OpenStack (no Nova, no Cinder)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + OVN                                 │
│  OVS (Open vSwitch) │ OVN (Open Virtual Network)               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  K8s nodes │ Neutron network nodes │ 10GbE NIC                 │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Using OpenStack Neutron's rich networking (FIPs, security groups, LBaaS) for Kubernetes pod networking without running full OpenStack. Useful when migrating from OpenStack to K8s while retaining network infrastructure.