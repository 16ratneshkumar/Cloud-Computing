[← Back to Index](../index.md)

# DEPLOYMENT 105 — Bare Metal → Kubernetes → Multi-CNI + SR-IOV → NFV Pods

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                    NFV Pods                    │
├──────────────────────────────────────────────────┤
│               Multi-CNI + SR-IOV               │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              NFV / CNF WORKLOADS                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  vRouter CNF Pod (requires 3 network interfaces):        │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  eth0: management net (default K8s CNI - Calico)   │  │   │
│  │  │  net1: SR-IOV VF (data plane - line rate DPDK)     │  │   │
│  │  │  net2: SR-IOV VF (data plane - second interface)   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES WITH MULTUS CNI                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Multus CNI (meta-plugin — enables multiple NICs per pod)│   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  NetworkAttachmentDefinition CRD                   │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  Primary: Calico CNI (management traffic)     │  │  │   │
│  │  │  │  Secondary: SR-IOV device plugin              │  │  │   │
│  │  │  │  (allocates VFs from physical NIC to pods)   │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  SR-IOV Device Plugin (DaemonSet — manages VF allocation) │   │
│  │  CPU Manager (pin vRouter pod CPUs to specific cores)    │   │
│  │  HugePages (for DPDK memory in vRouter pod)              │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              HOST OS PERFORMANCE LAYER                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  SR-IOV NIC: 16+ VFs per physical port                  │   │
│  │  DPDK (inside pod, user-space network I/O)               │   │
│  │  HugePages: 1GB pages for DPDK memory pool               │   │
│  │  CPU isolation: cores reserved for DPDK poll mode driver │   │
│  │  IOMMU (VT-d): required for SR-IOV VF passthrough        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL + IOMMU                               │
│  VFIO (Virtual Function I/O) driver │ SR-IOV kernel support    │
│  intel_iommu=on │ isolcpus= kernel params                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU: Intel Xeon (DPDK-optimized, NUMA-aware)                  │
│  RAM: 256GB+ with HugePages configured                         │
│  NIC: Intel X710 / XXV710 / E810 (SR-IOV, 25GbE+)             │
│  BIOS: VT-x + VT-d enabled │ HugePages │ CPU frequency fixed   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Multus CNI enables pods to have multiple network interfaces (K8s default CNI + additional)
- NetworkAttachmentDefinition (CRD) defines each additional network's CNI plugin config
- SR-IOV Device Plugin discovers NIC VFs and advertises them as Kubernetes resources (`intel.com/intel_sriov_netdevice`)
- Pods request SR-IOV VFs via resource requests — Multus attaches them at pod creation
- Inside the pod, DPDK bypasses the kernel and polls the VF directly for line-rate performance

## Use Case
5G user plane functions (UPF), packet gateways, vRouter CNFs requiring line-rate packet processing (40Gbps+) while still managed by Kubernetes.

## Pros
- Kubernetes API for carrier-grade network functions
- SR-IOV for near-native NIC performance inside pods
- DPDK inside containers achieves line-rate
- Multus supports arbitrary number of interfaces per pod

## Cons
- BIOS and kernel tuning required on every node
- SR-IOV limits number of pods per NIC (typically 32-128 VFs)
- DPDK in pods requires CPU core pinning — reduces scheduling flexibility
- Complex NetworkAttachmentDefinition configuration