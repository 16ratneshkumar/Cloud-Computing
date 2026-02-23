[← Back to Index](../index.md)

# DEPLOYMENT 42 — Bare Metal → OpenShift → Containers + KubeVirt VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│           Containers + KubeVirt VMs            │
├──────────────────────────────────────────────────┤
│                   OpenShift                    │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              MANAGEMENT PLANE                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenShift Console (web UI)                              │   │
│  │  oc CLI (OpenShift-extended kubectl)                     │   │
│  │  OpenShift GitOps (ArgoCD) │ OpenShift Pipelines (Tekton)│   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              RED HAT OPENSHIFT PLATFORM                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OCP Control Plane (RHCOS-based)                         │   │
│  │  kube-apiserver │ etcd │ scheduler │ controllers          │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CONTAINER WORKLOADS      VM WORKLOADS (OCP Virtualization│   │
│  │  ┌──────────────────┐     ┌──────────────────────────────┐│   │
│  │  │  App Pods         │     │  KubeVirt VMIs               ││   │
│  │  │  (standard OCI)   │     │  ┌───────────────────────┐  ││   │
│  │  └──────────────────┘     │  │  Windows Server VM    │  ││   │
│  │                           │  │  RHEL 9 VM            │  ││   │
│  │                           │  │  Legacy App VM        │  ││   │
│  │                           │  └───────────────────────┘  ││   │
│  │                           └──────────────────────────────┘│   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OPENSHIFT OPERATORS (key ones)                          │   │
│  │  ┌────────────────┐  ┌──────────────┐  ┌─────────────┐  │   │
│  │  │  OCP Virt      │  │  ODF         │  │  OCP        │  │   │
│  │  │  Operator      │  │  (OpenShift  │  │  Networking │  │   │
│  │  │  (KubeVirt)    │  │   Data       │  │  (OVN-K8s)  │  │   │
│  │  │                │  │  Foundation) │  │             │  │   │
│  │  └────────────────┘  └──────────────┘  └─────────────┘  │   │
│  │  ┌────────────────┐  ┌──────────────┐                   │   │
│  │  │  Cert Manager  │  │  Monitoring  │                   │   │
│  │  │  Operator      │  │  (Prometheus │                   │   │
│  │  │                │  │  + Grafana)  │                   │   │
│  │  └────────────────┘  └──────────────┘                   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  SECURITY LAYER                                          │   │
│  │  SELinux (enforcing) │ SCC (Security Context Constraints)│   │
│  │  NetworkPolicy │ OPA/Gatekeeper │ RBAC │ Pod Security    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              RHCOS (Red Hat CoreOS — immutable OS)              │
│  CRI-O (container runtime) │ KVM │ FIPS-140 crypto              │
│  Machine Config Operator manages OS updates                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  RHEL-certified hardware │ CPU (VT-x/AMD-V) │ RAM │ NVMe │ NIC │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- OpenShift is Red Hat's enterprise Kubernetes distribution based on RHCOS (immutable OS)
- OpenShift Virtualization operator installs KubeVirt and CDI for VM management
- ODF (OpenShift Data Foundation = Rook-Ceph) provides storage for both containers and VM disks
- OVN-Kubernetes provides SDN with NetworkPolicy enforcement
- Machine Config Operator manages RHCOS node configuration declaratively

## Use Case
Enterprise organizations standardizing on Red Hat with full support contract. Financial services, healthcare, government wanting to replace VMware while gaining Kubernetes.

## Pros
- Red Hat enterprise support for VMs + containers
- SELinux hardening throughout
- GitOps-native (ArgoCD included)
- ODF provides integrated HCI storage

## Cons
- Very expensive licensing
- High complexity
- OpenShift-specific conventions
- Heavy resource requirements