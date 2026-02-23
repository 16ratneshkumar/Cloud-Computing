[← Back to Index](../index.md)

# DEPLOYMENT 49 — Bare Metal → OpenStack → VMs → Kubernetes

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                   OpenStack                    │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes Pods (application workloads)                 │   │
│  │  PVCs backed by Cinder volumes (OpenStack storage)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (running inside OpenStack VMs)          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Control Plane VMs (Nova instances)  Worker VMs          │   │
│  │  ┌─────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │ kube-apiserver   │  │  kubelet │ containerd        │  │   │
│  │  │ etcd │ scheduler│  │  K8s pods                    │  │   │
│  │  └─────────────────┘  └──────────────────────────────┘  │   │
│  │  cloud-provider-openstack (integrates K8s with OpenStack)│   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Cinder CSI driver → Cinder volumes as PVCs       │  │   │
│  │  │  Octavia CCM → OpenStack LB as K8s LoadBalancer   │  │   │
│  │  │  Neutron CNI → Kuryr (K8s pods on Neutron nets)   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s nodes = Nova VMs
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK NOVA (compute)                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Nova VMs (K8s nodes) with appropriate flavors           │   │
│  │  Neutron networks for K8s node connectivity              │   │
│  │  Cinder volumes for K8s persistent storage               │   │
│  │  Octavia load balancers for K8s service endpoints        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM COMPUTE + LINUX KERNEL                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  OpenStack compute/storage/network/controller nodes             │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Traditional enterprise cloud pattern: OpenStack provides IaaS, Kubernetes runs on top. Used by telcos/enterprises that adopted OpenStack first and later added K8s.

## Pros
- Leverage existing OpenStack investment
- VM isolation for K8s nodes
- OpenStack services (Cinder/Octavia) integrate with K8s

## Cons
- Two complex systems to operate
- Double overhead (OpenStack VM + K8s pod)
- Most orgs are simplifying away from this pattern