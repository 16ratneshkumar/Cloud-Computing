[← Back to Index](../index.md)

# DEPLOYMENT 39 — Bare Metal → Kubernetes → Kata Containers → MicroVMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                    MicroVMs                    │
├──────────────────────────────────────────────────┤
│                Kata Containers                 │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Pods with Kata runtime (each pod = its own VM kernel)   │   │
│  │  ┌─────────────────────┐  ┌─────────────────────────┐   │   │
│  │  │  Pod A (Kata)        │  │  Pod B (runc normal)    │   │   │
│  │  │  ┌─────────────────┐ │  │  (regular container)    │   │   │
│  │  │  │  Container App  │ │  │  ┌──────────────────┐   │   │   │
│  │  │  │  (inside VM)    │ │  │  │  Container App   │   │   │   │
│  │  │  └─────────────────┘ │  │  └──────────────────┘   │   │   │
│  │  │  Own Linux kernel!   │  │  Shared host kernel      │   │   │
│  │  └─────────────────────┘  └─────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + containerd                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  containerd (CRI)                                        │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  RuntimeClass: kata-qemu  →  kata-qemu shim      │    │   │
│  │  │  RuntimeClass: runc       →  runc (standard)     │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  kata-runtime creates VM per pod
┌────────────────────────────▼────────────────────────────────────┐
│              KATA CONTAINERS RUNTIME                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kata-runtime (shim v2)                                  │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Per-Pod MicroVM:                                │    │   │
│  │  │  ┌────────────────────────────────────────────┐  │    │   │
│  │  │  │  QEMU / Firecracker / Cloud Hypervisor     │  │    │   │
│  │  │  │  (hypervisor backend, selectable)           │  │    │   │
│  │  │  │  ┌──────────────────────────────────────┐  │  │    │   │
│  │  │  │  │  Kata agent (inside VM)              │  │  │    │   │
│  │  │  │  │  → receives OCI commands             │  │  │    │   │
│  │  │  │  │  → starts container process in VM    │  │  │    │   │
│  │  │  │  └──────────────────────────────────────┘  │  │    │   │
│  │  │  │  Minimal kernel (kata-kernel) in VM        │  │    │   │
│  │  │  └────────────────────────────────────────────┘  │    │   │
│  │  └──────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL (host)                          │
│  kvm.ko │ kvm-intel/amd.ko │ QEMU/Firecracker per pod VM       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V mandatory) │ RAM │ NVMe │ NIC                 │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- `RuntimeClass: kata-qemu` tells containerd to use Kata runtime instead of runc
- Kata runtime creates a lightweight VM (QEMU/Firecracker backend) per Pod
- The Kata agent runs inside the VM, receives OCI container start commands from the host shim
- Container process runs inside the VM with its own kernel — full hardware isolation
- From Kubernetes perspective, it's just a pod — standard scheduling and networking

## Use Case
Multi-tenant Kubernetes where tenants' containers must not share a kernel (security requirement). Cloud providers offering shared K8s clusters to untrusted users.

## Pros
- Hardware isolation per pod (own kernel per pod)
- Standard Kubernetes API — no change to deployments
- Supports QEMU, Firecracker, and Cloud Hypervisor backends
- Better security than shared-kernel containers

## Cons
- Higher overhead per pod (VM boot + VM overhead)
- Slower pod startup time
- Some Kubernetes features limited (e.g., exec into container slower)
- Not all CSI drivers compatible