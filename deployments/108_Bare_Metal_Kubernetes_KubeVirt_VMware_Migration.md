[← Back to Index](../index.md)

# DEPLOYMENT 108 — Bare Metal → Kubernetes + KubeVirt → VMware Migration

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                VMware Migration                │
├──────────────────────────────────────────────────┤
│             Kubernetes + KubeVirt              │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKLOADS (VMware VMs migrated to KubeVirt)        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  VMs originally from VMware (Windows/Linux)              │   │
│  │  Now running as KubeVirt VMIs in Kubernetes              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + KUBEVIRT                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MTV (Migration Toolkit for Virtualization)               │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  MTV Operator (Red Hat / community)                │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  Provider: vSphere → K8s+KubeVirt            │  │  │   │
│  │  │  │  VDDK (VMware Data Disk Kit) → import VMDK   │  │  │   │
│  │  │  │  CDI imports disk → Ceph RBD PVC             │  │  │   │
│  │  │  │  VirtualMachine object created               │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  KubeVirt VMs (post-migration)                           │   │
│  │  virtctl console / virtctl ssh → access migrated VMs    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              SOURCE: VMWARE vSPHERE (during migration)          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ESXi hosts running existing VMs                         │   │
│  │  VMDK disk images (being read by MTV)                    │   │
│  │  vCenter API (source of VM inventory for MTV)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (target)                       │
│  New Kubernetes worker nodes                                    │
│  CPU (VT-x/AMD-V) │ RAM │ Ceph (VM disk storage) │ 25GbE NIC  │
└─────────────────────────────────────────────────────────────────┘

MIGRATION FLOW:
vCenter inventory → MTV discovers VMs → MTV copies VMDK (via VDDK)
→ CDI imports to Ceph RBD PVC → KubeVirt VMI created → VM boots
→ Validate → cutover (source VM powered off) → migration complete
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- MTV (Migration Toolkit for Virtualization) connects to vCenter as a "Provider"
- VDDK allows MTV to read VMware VMDK disk images
- CDI (Containerized Data Importer) imports the disk image into a Ceph RBD PVC
- KubeVirt creates a VirtualMachine object pointing to the imported PVC
- VM boots in KubeVirt — testing phase; then cutover (power off source, run target permanently)

## Use Case
Organizations migrating off VMware after Broadcom's 2024 price increases. Red Hat OpenShift Virtualization is the main commercial implementation.

## Pros
- Automated VMware-to-KubeVirt migration
- Minimal manual intervention
- Existing VMware admins can use familiar VM concepts in KubeVirt
- Open source (MTV is community + Red Hat)

## Cons
- VDDK has complex licensing
- Large VM disks = long migration time
- Network compatibility between VMware and KubeVirt networking requires planning
- Windows VMs need driver changes post-migration