[← Back to Index](../index.md)

# DEPLOYMENT 97 — Bare Metal → OpenNebula + K8s + KubeVirt + Ceph (Full)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│   OpenNebula + K8s + KubeVirt + Ceph (Full)    │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenNebula VMs (traditional IaaS tenants)               │   │
│  │  Kubernetes pods (via OneKE — OpenNebula K8s Engine)     │   │
│  │  KubeVirt VMs (inside K8s)                               │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENNEBULA (oned)                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Sunstone (web UI) │ onevm CLI │ Terraform provider      │   │
│  │  OneKE (K8s cluster deployment via ONE templates)        │   │
│  │  Scheduler │ VM lifecycle │ image/network management     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (OneKE-provisioned) + KUBEVIRT          │
│  KubeVirt for VM-in-K8s │ Rook-Ceph for K8s storage           │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              HYPERVISOR NODES (KVM via ONE)                     │
│  KVM + libvirt │ Ceph RBD client (VM disks) │ OpenNebula agent │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              CEPH STORAGE CLUSTER                               │
│  RBD (VM + K8s disks) │ RGW (object) │ CephFS (shared)        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Compute + storage nodes │ CPU │ RAM │ HDDs/NVMe │ 10GbE       │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
European research clouds (EGI, GÉANT ecosystem) wanting an OpenStack-lighter alternative with full VM + K8s + storage capabilities.