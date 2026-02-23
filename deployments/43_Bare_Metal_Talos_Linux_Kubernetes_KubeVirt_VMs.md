[← Back to Index](../index.md)

# DEPLOYMENT 43 — Bare Metal → Talos Linux → Kubernetes → KubeVirt VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                  KubeVirt VMs                  │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                  Talos Linux                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT CLIENTS                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  talosctl (Talos API client)  │  kubectl (K8s API)       │   │
│  │  NO SSH │ NO SHELL on nodes  │  virtctl (KubeVirt)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Talos API (gRPC, mTLS port 50000)
┌────────────────────────────▼────────────────────────────────────┐
│              TALOS LINUX (immutable OS)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  machined (PID 1 — Talos init, API server)               │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Everything is a static pod or system extension    │  │   │
│  │  │  No package manager │ No shell │ Read-only /usr   │  │   │
│  │  │  Configuration via talosctl apply-config           │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KUBERNETES (bootstrapped by Talos)                      │   │
│  │  kube-apiserver │ etcd │ scheduler │ kubelet              │   │
│  │  containerd (container runtime)                          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  CONTAINER PODS                                    │  │   │
│  │  │  App pods, system pods (Cilium CNI, etc.)          │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  KUBEVIRT VMs (via KubeVirt operator)              │  │   │
│  │  │  virt-launcher pods → QEMU processes               │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (Talos-specific build)                │
│  Minimal kernel │ KVM module │ CIS hardened │ no debug symbols  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V) │ RAM │ NVMe │ NIC                          │
│  PXE/iPXE boot │ No per-node manual configuration              │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Security-first Kubernetes where the OS has zero shell access. Air-gapped environments, compliance-focused deployments, and organizations treating nodes as pure API-managed infrastructure.

## Pros
- No shell/SSH — minimal attack surface
- Immutable OS — zero drift possible
- CIS-compliant by default
- API-only management (excellent for automation)
- Fast node reprovisioning via PXE

## Cons
- Steep learning curve for traditional sysadmins
- Debugging requires container-based tooling
- KubeVirt on Talos has less documentation
- Very different from conventional Linux admin