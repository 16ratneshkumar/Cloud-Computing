[← Back to Index](../index.md)

# DEPLOYMENT 88 — Bare Metal → Flatcar Container Linux → K8s

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      K8s                       │
├──────────────────────────────────────────────────┤
│            Flatcar Container Linux             │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container pods │ System pods                            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              FLATCAR CONTAINER LINUX                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  (CoreOS Container Linux successor — maintained by SUSE) │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Immutable read-only /usr (verified dm-verity)     │  │   │
│  │  │  A/B partition update system (Nebraska/Omaha)      │  │   │
│  │  │  systemd (PID 1) — limited to services             │  │   │
│  │  │  SSH access available (unlike Talos)               │  │   │
│  │  │  No package manager (static OS)                    │  │   │
│  │  │  Container-first: Docker, containerd built-in      │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Ignition / Butane (provisioning config at first   │  │   │
│  │  │  boot — NIC config, hostname, k8s join token, etc.)│  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Kubernetes (kubeadm / k3s / RKE2 installed atop Flatcar)│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (Flatcar hardened)                    │
│  systemd │ containerd │ dm-verity (root FS verification)        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  x86_64 / ARM64 │ BIOS/UEFI │ PXE boot or disk image          │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Immutable-OS Kubernetes nodes for organizations that want CoreOS-style stability, SUSE backing, and SSH access (unlike Talos). Popular for managed K8s node pools.

## Pros
- Immutable OS — zero drift possible
- A/B atomic updates (auto-update or controlled)
- SUSE commercial support
- SSH accessible (easier than Talos for ops teams)
- Used by Kinvolk (SUSE acquisition), Microsoft AKS nodes

## Cons
- Less restrictive than Talos (SSH exists)
- Smaller ecosystem than Ubuntu/RHEL for node OS
- Some software expects traditional package manager