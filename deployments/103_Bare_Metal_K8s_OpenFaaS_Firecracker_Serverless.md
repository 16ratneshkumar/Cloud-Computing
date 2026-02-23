[← Back to Index](../index.md)

# DEPLOYMENT 103 — Bare Metal → K8s → OpenFaaS + Firecracker → Serverless

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Serverless                   │
├──────────────────────────────────────────────────┤
│             OpenFaaS + Firecracker             │
├──────────────────────────────────────────────────┤
│                      K8s                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              FUNCTION INVOCATIONS                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  REST → Gateway → Function (Python/Go/Node.js)           │   │
│  │  Each invocation: Firecracker MicroVM start → exec → end │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + OpenFaaS + Kata-Firecracker           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  faas-netes (OpenFaaS K8s backend)                       │   │
│  │  Functions deployed as RuntimeClass: kata-firecracker    │   │
│  │  Auto-scale to zero / scale up based on Prometheus       │   │
│  │  NATS queue for async invocations                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KATA RUNTIME → FIRECRACKER VMM                     │
│  kata-runtime shim → firecracker binary → MicroVM per pod      │
│  jailer (cgroup + seccomp sandbox around Firecracker)           │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V) │ RAM (density: 1000s of MicroVMs) │ NVMe    │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.
