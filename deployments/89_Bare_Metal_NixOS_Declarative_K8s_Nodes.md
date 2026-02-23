[← Back to Index](../index.md)

# DEPLOYMENT 89 — Bare Metal → NixOS → Declarative K8s Nodes

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│             Declarative K8s Nodes              │
├──────────────────────────────────────────────────┤
│                     NixOS                      │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Pods │ services │ all K8s workloads                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              NixOS (declarative OS)                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  configuration.nix or flake.nix (declares EVERYTHING)   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  services.kubernetes.enable = true;                │  │   │
│  │  │  services.kubernetes.roles = ["master" "node"];    │  │   │
│  │  │  networking.firewall.allowedPorts = [6443 ...];    │  │   │
│  │  │                                                    │  │   │
│  │  │  Nix package manager: 80,000+ reproducible pkgs   │  │   │
│  │  │  nixos-rebuild switch → atomic OS reconfig        │  │   │
│  │  │  nixos-rebuild rollback → revert any change       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  REPRODUCIBILITY:                                  │  │   │
│  │  │  Same configuration.nix → identical system state  │  │   │
│  │  │  anywhere (like Dockerfile but for entire OS)     │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (NixOS-managed)                       │
│  Nix store: /nix/store (immutable, content-addressed packages)  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  x86_64 / ARM64 │ Any hardware with Linux driver support       │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Research teams and DevOps practitioners wanting fully reproducible, declarative OS + K8s node configuration as code. Gaining traction in tech companies with strong infrastructure-as-code culture.

## Pros
- Fully declarative — config file = complete system description
- Atomic upgrades and rollbacks
- Reproducible builds (same config = identical system)
- 80,000+ packages in nixpkgs

## Cons
- Steep learning curve (Nix language is unusual)
- Small community vs Ubuntu/RHEL
- Some proprietary software hard to package
- Not common in enterprise