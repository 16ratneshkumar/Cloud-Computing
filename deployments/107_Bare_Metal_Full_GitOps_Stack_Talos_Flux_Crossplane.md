[← Back to Index](../index.md)

# DEPLOYMENT 107 — Bare Metal → Full GitOps Stack (Talos + Flux + Crossplane)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│ Full GitOps Stack (Talos + Flux + Crossplane)  │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              EVERYTHING-AS-CODE (Git is source of truth)        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Git repositories (GitLab / GitHub)                      │   │
│  │  infra-repo/      app-repo/       platform-repo/         │   │
│  │  (Talos config)   (K8s manifests) (Crossplane XRDs)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  GitOps reconciliation (push/pull)
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + GITOPS LAYER                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  FluxCD (GitOps operator)                                │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  source-controller → watch Git repo               │  │   │
│  │  │  kustomize-controller → apply Kustomize manifests │  │   │
│  │  │  helm-controller → manage Helm chart releases      │  │   │
│  │  │  notification-controller → alerts + webhooks       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Crossplane (Infrastructure as Code inside K8s)          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  XRD (Composite Resource Definitions)              │  │   │
│  │  │  → provision cloud resources as K8s CRDs           │  │   │
│  │  │  → databases, DNS, S3 buckets via kubectl apply    │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  APPLICATION WORKLOADS (reconciled by Flux)             │   │
│  │  Pods │ Services │ Ingress │ PVCs                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  nodes managed by Talos API
┌────────────────────────────▼────────────────────────────────────┐
│              TALOS LINUX NODES (immutable OS)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  machined (API server) │ kubelet │ containerd             │   │
│  │  talosctl apply-config (from infra-repo in Git)          │   │
│  │  A/B OS partition upgrade (from infra-repo pipeline)     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (Talos) + HARDWARE                    │
└────────────────────────────┬────────────────────────────────────┘
                             │  Metal3/iPXE for initial provisioning
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  PXE/iPXE boot → Talos installer from Git-triggered pipeline   │
│  CPU │ RAM │ NVMe │ NIC │ BMC (Redfish for Metal3 control)     │
└─────────────────────────────────────────────────────────────────┘

WORKFLOW: Git commit → CI pipeline → Flux reconciles → cluster
          changes applied → no human touch after initial boot
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Talos config (machineconfig YAML) lives in infra-repo → any node config change is a Git PR
- Flux watches app-repo and applies K8s manifests automatically on every commit
- Crossplane provisions external infrastructure (DNS, DBs) via K8s CRDs — all in Git
- Metal3 provisions new bare metal nodes from Git-defined BareMetalHost resources
- Zero manual node access: no SSH, no kubectl on dev laptops in production

## Use Case
Cloud-native-first teams and SRE platforms wanting zero-touch infrastructure with full auditability and reproducibility. Growing adoption in fintech and security-sensitive environments.

## Pros
- 100% Git-driven — every change is auditable
- No snowflake servers — any node can be rebuilt from Git
- Crossplane enables cloud + bare metal from same Git repo
- Flux handles app deployment + infrastructure

## Cons
- Steep learning curve (Talos + Flux + Crossplane simultaneously)
- Bootstrap chicken-and-egg problem (need running K8s to run Flux)
- Crossplane providers still maturing
- Debugging complex reconciliation chains