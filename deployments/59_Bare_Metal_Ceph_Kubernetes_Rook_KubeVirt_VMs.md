[← Back to Index](../index.md)

# DEPLOYMENT 59 — Bare Metal → Ceph + Kubernetes (Rook) + KubeVirt → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│      Ceph + Kubernetes (Rook) + KubeVirt       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container Pods (PVCs → Ceph RBD / CephFS)               │   │
│  │  KubeVirt VMs (DataVolumes → Ceph RBD)                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES                                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ROOK OPERATOR (manages Ceph as K8s resources)           │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  CephCluster CRD → creates/manages Ceph cluster    │  │   │
│  │  │  CephBlockPool CRD → Ceph RBD pool for PVCs        │  │   │
│  │  │  CephFilesystem CRD → CephFS for shared PVCs       │  │   │
│  │  │  CephObjectStore CRD → S3/Swift API                │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Rook-Ceph CSI Driver (provisioner + attacher)     │  │   │
│  │  │  → dynamic PVC provisioning from Ceph RBD          │  │   │
│  │  │  → ReadWriteMany PVCs via CephFS                   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  KUBEVIRT OPERATOR                                 │  │   │
│  │  │  VMs with DataVolumes backed by Ceph RBD PVCs      │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Rook deploys Ceph pods on nodes
┌────────────────────────────▼────────────────────────────────────┐
│              CEPH CLUSTER (managed by Rook in K8s)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ceph-mon pods (3+) │ ceph-mgr pods │ ceph-osd pods      │   │
│  │  (each OSD pod = one physical disk on a node)            │   │
│  │  ceph-mds pods (CephFS) │ ceph-rgw pods (object)         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (each node)                     │
│  kvm.ko │ containerd │ direct disk access for OSD pods          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Converged nodes (K8s + Ceph on same hardware):                 │
│  CPU 32c+ │ RAM 256GB+ │ NVMe (for K8s) + HDDs (for Ceph OSDs) │
│  25GbE NIC (shared cluster + storage traffic)                   │
│  Minimum 3 nodes for Ceph quorum                                │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Rook Operator runs inside Kubernetes and manages Ceph cluster lifecycle via CRDs
- Rook deploys ceph-mon, ceph-mgr, ceph-osd pods on K8s nodes — each OSD pod claims a raw disk
- Rook-Ceph CSI driver enables dynamic PVC provisioning: `kubectl create pvc` → RBD volume in Ceph
- KubeVirt DataVolumes use Ceph RBD PVCs as VM boot disks — live migration moves VM + Ceph handles replication
- This creates a fully Kubernetes-native HCI stack (Harvester uses this architecture)

## Use Case
Cloud-native HCI — open-source alternative to Nutanix/vSAN. Organizations wanting one K8s control plane for VMs, containers, and storage.

## Pros
- Fully K8s-native — Ceph managed as K8s CRDs
- Dynamic storage provisioning for both containers and VMs
- Self-healing distributed storage
- No OpenStack dependency

## Cons
- Rook-Ceph adds K8s operational complexity
- Ceph tuning knowledge still required
- Minimum 3 nodes for quorum
- VM disk latency from RBD can be higher than local NVMe