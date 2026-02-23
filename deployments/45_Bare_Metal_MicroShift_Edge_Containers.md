[← Back to Index](../index.md)

# DEPLOYMENT 45 — Bare Metal → MicroShift → Edge Containers

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                Edge Containers                 │
├──────────────────────────────────────────────────┤
│                   MicroShift                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              EDGE APPLICATIONS                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Container Pods (edge workloads)                         │   │
│  │  RHEL device agents │ AI/ML inference │ IoT data collect  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              MicroShift (minimal OpenShift runtime)             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Embedded OpenShift API components (~200MB RAM baseline) │   │
│  │  kube-apiserver │ controller-manager │ etcd              │   │
│  │  CRI-O (container runtime)                               │   │
│  │  OVN-Kubernetes (CNI — simplified)                       │   │
│  │  CSI (local path provisioner)                            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              RHEL 8/9 HOST OS                                   │
│  CRI-O │ KVM (optional for VM support) │ SELinux enforcing      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Edge Device)                  │
│  Industrial PC / Raspberry Pi 4+ / Edge server                  │
│  ARM64 or x86_64 │ 2GB+ RAM │ 16GB+ storage                    │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Red Hat's minimal OpenShift for edge devices — point-of-sale, factory floor, vehicle compute, and medical devices with Red Hat support.

## Pros
- ~200MB RAM baseline — very small footprint
- OpenShift API compatibility
- Works offline/disconnected
- Red Hat support

## Cons
- Very new product
- Limited production track record
- Red Hat subscription required