[← Back to Index](../index.md)

# DEPLOYMENT 48 — Bare Metal → OpenStack → Magnum → Kubernetes Clusters

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│              Kubernetes Clusters               │
├──────────────────────────────────────────────────┤
│                     Magnum                     │
├──────────────────────────────────────────────────┤
│                   OpenStack                    │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT INTERFACES                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  openstack coe cluster create myCluster                  │   │
│  │  kubectl (against provisioned cluster)                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK MAGNUM (Kubernetes-as-a-Service)         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  magnum-api  │  magnum-conductor                         │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  ClusterTemplate (defines: K8s ver, image, flavor) │  │   │
│  │  │  Cluster object → provisions K8s via Heat template │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  Heat Stack → Nova VMs for K8s nodes         │  │  │   │
│  │  │  │  cloud-init → runs kubeadm on each VM        │  │  │   │
│  │  │  │  Neutron → K8s node networks + LB            │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Magnum calls Nova/Neutron/Cinder
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK INFRASTRUCTURE SERVICES                  │
│  Nova (VMs for K8s nodes) │ Neutron (K8s networking)            │
│  Cinder (persistent volumes) │ Octavia (K8s LoadBalancer)       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM COMPUTE NODES + LINUX KERNEL                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  K8s Control Plane VMs    K8s Worker VMs                 │   │
│  │  (provisioned by Magnum)  (provisioned by Magnum)        │   │
│  │  ┌────────────────┐       ┌────────────────────────────┐ │   │
│  │  │ kube-apiserver │       │  kubelet │ containerd       │ │   │
│  │  │ etcd │ scheduler│      │  K8s pods (tenant apps)    │ │   │
│  │  └────────────────┘       └────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Compute nodes │ Storage nodes │ Network nodes │ Controllers    │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Kubernetes-as-a-Service for tenants of an OpenStack private cloud — self-service K8s cluster provisioning within an existing OpenStack environment.

## Pros
- K8s clusters as first-class OpenStack resources
- Self-service for tenants
- Integrates with OpenStack auth, networking, storage

## Cons
- Complex (OpenStack + Magnum + K8s)
- Slow cluster provisioning
- Magnum has limited development momentum
- Many failure points