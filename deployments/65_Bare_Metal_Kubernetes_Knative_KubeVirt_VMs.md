[← Back to Index](../index.md)

# DEPLOYMENT 65 — Bare Metal → Kubernetes → Knative → KubeVirt VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                  KubeVirt VMs                  │
├──────────────────────────────────────────────────┤
│                    Knative                     │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              EVENT-DRIVEN VM WORKLOADS                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Legacy App VM (scales to zero when idle)                │   │
│  │  KubeVirt VMI triggered by Knative event                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  (experimental)
┌────────────────────────────▼────────────────────────────────────┐
│              KNATIVE SERVING + EVENTING                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Knative Serving → scale-to-zero for K8s workloads       │   │
│  │  Knative Eventing → CloudEvents trigger VM start         │   │
│  │  (KubeVirt integration: experimental/community built)    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + KUBEVIRT                              │
│  kube-apiserver │ KubeVirt operator │ virt-launcher pods        │
│  QEMU processes (one per VM)                                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM + LINUX KERNEL                                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Serverless scaling for VM workloads — experimental pattern for legacy apps that can't be containerized but need event-driven provisioning.

## Pros
- Scale-to-zero for VM workloads
- Event-driven provisioning
- Kubernetes-native

## Cons
- Highly experimental — not production-ready
- VM cold starts are slow (seconds to minutes)
- Limited documentation and community examples
- Knative + KubeVirt integration is not well-maintained