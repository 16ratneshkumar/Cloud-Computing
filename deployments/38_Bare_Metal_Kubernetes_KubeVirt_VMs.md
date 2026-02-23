[← Back to Index](../index.md)

# DEPLOYMENT 38 — Bare Metal → Kubernetes → KubeVirt → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                    KubeVirt                    │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES API CLIENTS                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  kubectl apply -f vm.yaml   virtctl (KubeVirt CLI plugin)  │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES CONTROL PLANE                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kube-apiserver │ etcd │ kube-scheduler │ controllers     │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KUBEVIRT CONTROL PLANE (custom resource controllers)    │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  virt-controller (watches VM/VMI objects)          │  │   │
│  │  │  virt-api (webhook admission controller)           │  │   │
│  │  │  virt-operator (installs/upgrades KubeVirt)        │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  KUBEVIRT CRDs:                                           │   │
│  │  VirtualMachine (VM) │ VirtualMachineInstance (VMI)      │   │
│  │  VirtualMachineInstanceMigration │ DataVolume (CDI)      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  schedules VMI as a Pod
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES WORKER NODE                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CONTAINER PODS (regular)     VM PODS (KubeVirt)         │   │
│  │  ┌──────────────┐             ┌──────────────────────┐   │   │
│  │  │  App Pod A   │             │  virt-launcher Pod   │   │   │
│  │  │  (nginx)     │             │  ┌────────────────┐  │   │   │
│  │  └──────────────┘             │  │  QEMU process  │  │   │   │
│  │  ┌──────────────┐             │  │  (1 per VM)    │  │   │   │
│  │  │  App Pod B   │             │  │  KVM-backed    │  │   │   │
│  │  │  (redis)     │             │  │  VMI workload  │  │   │   │
│  │  └──────────────┘             │  └────────────────┘  │   │   │
│  │                               └──────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  virt-handler (DaemonSet — per node agent)               │   │
│  │  → manages QEMU processes on this node                   │   │
│  │  → handles live migration                                │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CDI (Containerized Data Importer)                       │   │
│  │  → imports disk images (qcow2/ISO) into PVCs             │   │
│  │  → provides DataVolume CRD for VM disk management        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              STORAGE LAYER (for VM disks)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  PVC (PersistentVolumeClaim) → DataVolume (CDI)          │   │
│  │  ┌───────────┐  ┌───────────┐  ┌──────────────────────┐ │   │
│  │  │ Ceph RBD  │  │ Longhorn  │  │ NFS / hostPath       │ │   │
│  │  │ (CSI)     │  │ (CSI)     │  │ (local dev)          │ │   │
│  │  └───────────┘  └───────────┘  └──────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (each node)                     │
│  kvm.ko │ kvm-intel/amd.ko │ QEMU process per VM               │
│  containerd (for regular pods) │ cgroups │ TUN/TAP              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V mandatory) │ RAM │ NVMe/SAN │ 25GbE NIC       │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- KubeVirt extends Kubernetes with CRDs: VirtualMachine, VirtualMachineInstance
- `kubectl apply -f vm.yaml` creates a VM object; virt-controller creates a virt-launcher Pod
- The virt-launcher Pod runs a QEMU process — the actual VM
- virt-handler (DaemonSet) manages QEMU on each node and coordinates live migration
- CDI imports disk images from external URLs/registries into PVCs for VM boot disks
- VMs get Kubernetes service discovery, RBAC, and network policies like any other workload

## Use Case
Organizations consolidating VM and container management under one Kubernetes control plane. Red Hat OpenShift Virtualization is the enterprise version. Used to migrate VMware VMs to Kubernetes.

## Pros
- Single control plane for VMs and containers
- kubectl/GitOps manages VMs
- Live migration via Kubernetes API
- Kubernetes RBAC applies to VMs
- Active CNCF project

## Cons
- Storage (CDI + PVC) setup is complex
- VM performance slightly lower than bare KVM
- Requires KubeVirt-specific expertise on top of K8s
- Debugging spans K8s + QEMU layers