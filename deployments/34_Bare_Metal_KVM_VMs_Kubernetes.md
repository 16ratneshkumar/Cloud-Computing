[← Back to Index](../index.md)

# DEPLOYMENT 34 — Bare Metal → KVM → VMs → Kubernetes

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                      KVM                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS (inside VMs)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes Cluster (e.g. cluster-prod-01)               │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Control Plane VMs           Worker VMs            │  │   │
│  │  │  ┌──────────────┐   ┌──────┐ ┌──────┐  ┌──────┐   │  │   │
│  │  │  │ kube-apiserver│   │wkr-1 │ │wkr-2 │  │wkr-3 │   │  │   │
│  │  │  │ etcd          │   │pods  │ │pods  │  │pods  │   │  │   │
│  │  │  │ scheduler     │   └──────┘ └──────┘  └──────┘   │  │   │
│  │  │  └──────────────┘                                   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  (Multiple isolated K8s clusters can run on same bare metal)    │
└────────────────────────────┬────────────────────────────────────┘
                             │  each K8s node = a VM
┌────────────────────────────▼────────────────────────────────────┐
│              VIRTUAL MACHINES (KVM guests)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VM: ctrl-1        VM: ctrl-2       VM: ctrl-3           │   │
│  │  (K8s ctrl plane)  (K8s ctrl plane) (K8s ctrl plane)     │   │
│  │  8vCPU / 16GB      8vCPU / 16GB     8vCPU / 16GB        │   │
│  │                                                          │   │
│  │  VM: wkr-1         VM: wkr-2        VM: wkr-3            │   │
│  │  (K8s worker)      (K8s worker)     (K8s worker)         │   │
│  │  16vCPU / 64GB     16vCPU / 64GB    16vCPU / 64GB       │   │
│  │  virtio-net        virtio-net       virtio-net           │   │
│  │  virtio-scsi       virtio-scsi      virtio-scsi          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  VM provisioning
┌────────────────────────────▼────────────────────────────────────┐
│              VM MANAGEMENT LAYER                                │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  Terraform       │  │  Ansible         │  │  libvirt /   │  │
│  │  (provision VMs) │  │  (configure K8s) │  │  virsh       │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              QEMU + KVM (hypervisor layer)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  One QEMU process per VM                                 │   │
│  │  KVM kernel module → hardware virtualization            │   │
│  │  Linux bridges / OVS → VM networking                    │   │
│  │  LVM / Ceph RBD → VM disk storage                       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + KVM MODULE (host)                   │
│  kvm.ko │ kvm-intel.ko / kvm-amd.ko │ TUN/TAP │ OVS            │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Host 1 (HV)        Host 2 (HV)        Host 3 (HV)      │   │
│  │  CPU: 64c/128t      CPU: 64c/128t      CPU: 64c/128t    │   │
│  │  RAM: 512GB         RAM: 512GB         RAM: 512GB        │   │
│  │  NVMe: 4x 4TB       NVMe: 4x 4TB      NVMe: 4x 4TB      │   │
│  │  NIC: 25GbE x2      NIC: 25GbE x2     NIC: 25GbE x2    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

MULTI-CLUSTER LAYOUT on same bare metal:
┌─────────────────────────────────────────────────────────────┐
│  BARE METAL HOST POOL                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  KVM (hypervisor)                                   │   │
│  │  ┌─────────────────┐  ┌─────────────────┐          │   │
│  │  │  K8s Cluster A  │  │  K8s Cluster B  │          │   │
│  │  │  (production)   │  │  (staging)      │          │   │
│  │  │  VMs: ctrl+wkr  │  │  VMs: ctrl+wkr  │          │   │
│  │  └─────────────────┘  └─────────────────┘          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Bare metal hosts run KVM hypervisor with libvirt management
- VMs are provisioned (via Terraform/Ansible) and each VM becomes a Kubernetes node
- Multiple Kubernetes clusters share the same physical hardware via VM isolation
- Each K8s cluster has full VM isolation — blast radius contained per cluster
- VM snapshots enable quick K8s node recovery

## Use Case
Most common enterprise Kubernetes deployment. Banks, retailers, telcos running multiple K8s clusters with VM-level isolation between them on shared hardware.

## Pros
- VM isolation between K8s clusters prevents cross-cluster impact
- Familiar VM operations (snapshot, backup, clone)
- Multiple clusters on same hardware
- Easy node replacement — redeploy VM

## Cons
- VM overhead reduces container density vs bare-metal K8s
- Two layers to manage (KVM + K8s)
- Higher per-pod latency than bare-metal K8s
- Storage I/O passes through VM layer