[← Back to Index](../index.md)

# DEPLOYMENT 87 — Bare Metal → Talos Linux → K8s (Pure Immutable OS)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│            K8s (Pure Immutable OS)             │
├──────────────────────────────────────────────────┤
│                  Talos Linux                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Application pods │ System pods (CNI, CSI, monitoring)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              TALOS LINUX (immutable OS)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  machined (PID 1, gRPC API server — replaces init/systemd│   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  No shell (no bash/sh)                             │  │   │
│  │  │  No SSH daemon                                     │  │   │
│  │  │  No package manager (apt/yum)                      │  │   │
│  │  │  No interactive login                              │  │   │
│  │  │  Read-only /usr filesystem (squashfs)              │  │   │
│  │  │  All config via talosctl (gRPC, mTLS)              │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  KUBERNETES (bootstrapped via Talos)               │  │   │
│  │  │  containerd → static pods → K8s control plane      │  │   │
│  │  │  kubelet → worker node agent                       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  OS UPDATE MODEL (A/B partition schema)            │  │   │
│  │  │  Partition A (current) │ Partition B (new version) │  │   │
│  │  │  talosctl upgrade → writes B, reboots into B       │  │   │
│  │  │  Rollback: if unhealthy → boot back to A           │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              TALOS LINUX KERNEL (custom minimal build)          │
│  Minimal config │ CIS hardened │ No debug symbols │ FIPS mode   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Any x86_64 / ARM64 hardware │ UEFI preferred                  │
│  PXE boot supported │ Cloud images (AWS/GCP/Azure/metal)        │
└─────────────────────────────────────────────────────────────────┘

TALOSCTL WORKFLOW (no SSH required):
┌─────────────────────────────────────────────────────────────────┐
│  Developer laptop → talosctl apply-config → node gRPC API       │
│  talosctl upgrade → A/B OS partition upgrade                    │
│  talosctl kubeconfig → get K8s credentials                      │
│  talosctl dmesg / talosctl logs → view logs (no SSH needed)     │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Talos boots from an immutable squashfs root — nothing can modify /usr
- machined (PID 1) replaces systemd/init and serves a gRPC API over mTLS
- All management is done via talosctl — no SSH, no shell, no interactive access
- K8s is bootstrapped from machined's control — kubelet and static pods start automatically
- OS upgrades: talosctl writes new OS image to inactive A/B partition and reboots

## Use Case
Security-critical Kubernetes environments needing zero shell attack surface. Compliance-heavy industries (fintech, healthcare), air-gapped deployments, and GitOps-everything teams.

## Pros
- Zero shell access — minimal attack surface
- CIS-hardened kernel by default
- A/B atomic upgrades — no manual patching
- FIPS 140-2 mode available
- GitOps-native

## Cons
- No traditional Linux admin tools (very different mental model)
- Debugging requires container-based ephemeral pods
- Smaller ecosystem than standard K8s distros
- Talos-specific toolchain learning curve