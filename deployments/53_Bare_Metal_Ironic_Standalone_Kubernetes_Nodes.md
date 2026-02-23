[← Back to Index](../index.md)

# DEPLOYMENT 53 — Bare Metal → Ironic Standalone → Kubernetes Nodes

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                Kubernetes Nodes                │
├──────────────────────────────────────────────────┤
│               Ironic Standalone                │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES CLUSTER (after provisioning)            │
│  kube-apiserver │ kubelet │ containerd │ Pods                   │
└────────────────────────────┬────────────────────────────────────┘
                             │  nodes provisioned by Ironic
┌────────────────────────────▼────────────────────────────────────┐
│              IRONIC STANDALONE (no full OpenStack)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ironic-api │ ironic-conductor │ ironic-inspector        │   │
│  │  (runs without Nova, Neutron, Keystone — minimal mode)   │   │
│  │  Standalone DHCP/TFTP for PXE boot                       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Physical servers with BMC (IPMI/Redfish) │ PXE-capable NICs   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Lightweight bare metal provisioning without full OpenStack complexity — just use Ironic to automate hardware for Kubernetes clusters.