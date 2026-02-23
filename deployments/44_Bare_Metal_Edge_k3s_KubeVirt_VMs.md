[← Back to Index](../index.md)

# DEPLOYMENT 44 — Bare Metal (Edge) → k3s → KubeVirt VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                  KubeVirt VMs                  │
├──────────────────────────────────────────────────┤
│                      k3s                       │
├──────────────────────────────────────────────────┤
│               Bare Metal (Edge)                │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              EDGE WORKLOADS                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container Pods (lightweight edge apps)                  │   │
│  │  KubeVirt VMs (legacy edge VMs, e.g. POS terminal OS)    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              k3s + KubeVirt (edge cluster)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  k3s (single binary K8s)                                 │   │
│  │  KubeVirt operator (installed on top of k3s)             │   │
│  │  CDI (Containerized Data Importer for VM disks)          │   │
│  │  Multus / Flannel (networking)                           │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (edge node)                     │
│  kvm.ko (requires VT-x/AMD-V in edge CPU) │ containerd          │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Edge Node)                    │
│  Intel NUC / industrial PC / server edge appliance              │
│  CPU (must support VT-x) │ RAM 16GB+ │ NVMe 256GB+              │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Edge locations running both containers (modern apps) and VMs (legacy apps) with minimal resource footprint. Telco edge, retail, manufacturing.

## Pros
- Lightweight K8s (k3s) + VM support at edge
- Single platform for mixed workloads

## Cons
- KubeVirt adds significant resource requirements to k3s
- Limited testing of this combination
- Edge hardware must have VT-x support