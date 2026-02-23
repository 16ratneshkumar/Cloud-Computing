[← Back to Index](../index.md)

# DEPLOYMENT 52 — Bare Metal → Kubernetes → Metal3 → Bare Metal Nodes

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                Bare Metal Nodes                │
├──────────────────────────────────────────────────┤
│                     Metal3                     │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              GITOPS / AUTOMATION CLIENTS                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kubectl apply -f baremetalhost.yaml                     │   │
│  │  ClusterAPI (CAPI) controller                            │   │
│  │  ArgoCD / FluxCD (GitOps)                                │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Kubernetes CRDs
┌────────────────────────────▼────────────────────────────────────┐
│              MANAGEMENT KUBERNETES CLUSTER                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Metal3 (baremetal-operator)                             │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  BareMetalHost CRD (represents a physical server)  │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  spec:                                       │  │  │   │
│  │  │  │    bmc:                                      │  │  │   │
│  │  │  │      address: redfish://10.0.0.1             │  │  │   │
│  │  │  │    image:                                    │  │  │   │
│  │  │  │      url: https://images/rhcos.qcow2         │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Ironic (runs as pods in K8s)                      │  │   │
│  │  │  ironic-api pod │ ironic-conductor pod             │  │   │
│  │  │  ironic-inspector pod                              │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Cluster API (CAPI) + CAPM3 provider              │  │   │
│  │  │  → provisions entire K8s clusters on bare metal   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Ironic → IPMI/Redfish BMC
┌────────────────────────────▼────────────────────────────────────┐
│              BARE METAL NODE POOL (target hardware)             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1                Node 2              Node 3        │   │
│  │  BMC: Redfish         BMC: Redfish         BMC: Redfish  │   │
│  │  ← Metal3/Ironic      ← Metal3/Ironic      ← Metal3      │   │
│  │    controls power,       controls              controls  │   │
│  │    boot order,           power+boot           power+boot │   │
│  │    OS image deploy       OS image             OS image   │   │
│  │                                                          │   │
│  │  After provisioning → become K8s worker nodes           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Metal3 introduces `BareMetalHost` CRD — physical servers become Kubernetes objects
- baremetal-operator reconciles BareMetalHost state — if `spec.online: true`, it powers on the node
- Ironic (running as K8s pods) handles the actual provisioning: BMC control, PXE boot, image deploy
- Cluster API (CAPI) with CAPM3 provider creates K8s clusters by provisioning BareMetalHosts
- Entire bare metal infrastructure is managed as YAML in Git — true GitOps for hardware

## Use Case
GitOps-driven bare metal Kubernetes provisioning. Foundation of OpenShift on bare metal and modern K8s cluster lifecycle management.

## Pros
- Bare metal as code (YAML BareMetalHost resources)
- GitOps-friendly
- Cluster API integration for automated cluster lifecycle
- Active CNCF community

## Cons
- Complex setup (Metal3 + Ironic + CAPI)
- Slow provisioning (~5-10 minutes per node)
- Requires out-of-band management (Redfish/IPMI)
- Limited advanced networking configuration