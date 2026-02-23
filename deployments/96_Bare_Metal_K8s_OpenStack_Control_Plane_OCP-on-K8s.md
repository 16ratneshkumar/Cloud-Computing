[← Back to Index](../index.md)

# DEPLOYMENT 96 — Bare Metal → K8s → OpenStack Control Plane (OCP-on-K8s)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│      OpenStack Control Plane (OCP-on-K8s)      │
├──────────────────────────────────────────────────┤
│                      K8s                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT VMs (OpenStack instances)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Tenant VMs provisioned by Nova API                      │   │
│  │  Running on bare metal compute nodes with KVM            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  nova-compute → libvirt → KVM on BM
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK SERVICES (as K8s pods)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Deployed via: OpenStack-Helm OR OpenStack Operator      │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  keystone deploy   │ keystone pod(s) in K8s      │    │   │
│  │  │  nova deploy       │ nova-api, nova-scheduler,   │    │   │
│  │  │                    │ nova-conductor pods          │    │   │
│  │  │  neutron deploy    │ neutron-server pods          │    │   │
│  │  │  cinder deploy     │ cinder-api, cinder-vol pods  │    │   │
│  │  │  glance deploy     │ glance-api pods              │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Infrastructure pods (StatefulSets):             │    │   │
│  │  │  MariaDB-Galera-0,1,2 │ RabbitMQ-0,1,2          │    │   │
│  │  │  Memcached-0,1,2      │ Ingress controller       │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s manages OpenStack pods
┌────────────────────────────▼────────────────────────────────────┐
│              MANAGEMENT KUBERNETES CLUSTER                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kube-apiserver │ etcd │ scheduler │ controllers          │   │
│  │  Ceph CSI (persistent storage for MariaDB, RabbitMQ)     │   │
│  │  Multus CNI (provider network access for Neutron)        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              BARE METAL COMPUTE NODES (outside K8s)             │
│  nova-compute │ neutron-agent │ libvirt │ KVM                   │
│  (join OpenStack but NOT managed by K8s)                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  K8s control nodes: CPU 32c │ RAM 256GB │ NVMe │ 25GbE         │
│  Compute nodes: CPU 128c │ RAM 1TB+ │ NVMe │ 25GbE×2          │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
AT&T, Verizon-style deployments where OpenStack is the tenant API but K8s manages the control plane for GitOps-driven operations and self-healing.