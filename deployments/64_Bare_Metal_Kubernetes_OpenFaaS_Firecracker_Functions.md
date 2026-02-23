[← Back to Index](../index.md)

# DEPLOYMENT 64 — Bare Metal → Kubernetes → OpenFaaS → Firecracker → Functions

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Functions                    │
├──────────────────────────────────────────────────┤
│                  Firecracker                   │
├──────────────────────────────────────────────────┤
│                    OpenFaaS                    │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              DEVELOPER / FUNCTION CLIENTS                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  faas-cli deploy --image myfunction:latest               │   │
│  │  curl https://gateway.example.com/function/myfunction    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENFAAS COMPONENTS (K8s Deployments)              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  faas-netes (K8s backend for OpenFaaS)                   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  API Gateway (faas-gateway pod)                    │  │   │
│  │  │  → receives HTTP invocations                       │  │   │
│  │  │  → routes to function pods                        │  │   │
│  │  │  → auto-scaling (scale to zero + scale up)        │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Prometheus (auto-scaling metrics)                 │  │   │
│  │  │  NATS (async function invocation queue)            │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  each function = a Pod with Firecracker runtime
┌────────────────────────────▼────────────────────────────────────┐
│              FUNCTION PODS (Firecracker-backed via Kata/runtime)│
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Function Pod A     Function Pod B     Function Pod N    │   │
│  │  RuntimeClass:      RuntimeClass:      RuntimeClass:     │   │
│  │  kata-firecracker   kata-firecracker   kata-firecracker  │   │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐  │   │
│  │  │ Firecracker  │   │ Firecracker  │   │ Firecracker  │  │   │
│  │  │ MicroVM      │   │ MicroVM      │   │ MicroVM      │  │   │
│  │  │ (Python fn)  │   │ (Node.js fn) │   │ (Go fn)      │  │   │
│  │  └──────────────┘   └──────────────┘   └──────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES + KVM + LINUX KERNEL                    │
│  containerd │ Kata runtime (kata-fc) │ KVM module              │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU (VT-x/AMD-V) │ RAM │ NVMe │ 10GbE NIC                    │
└─────────────────────────────────────────────────────────────────┘

INVOCATION FLOW:
developer → HTTP → Gateway pod → Route → Function pod
→ Kata shim → Firecracker MicroVM → function executes
→ return → Gateway → developer
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- OpenFaaS gateway receives HTTP function invocations
- faas-netes deploys function workloads as K8s pods with `RuntimeClass: kata-firecracker`
- Kata runtime uses Firecracker as its VM backend — each pod runs in a MicroVM
- Prometheus metrics drive HPA (Horizontal Pod Autoscaler) for scale-to-zero and scale-up
- NATS provides async invocation queuing for high-throughput scenarios

## Use Case
Self-hosted serverless platform for organizations with data residency requirements or cost savings vs AWS Lambda at scale.

## Pros
- True serverless on bare metal
- Strong Firecracker VM isolation per function
- Cost savings vs public cloud FaaS at scale
- Data residency compliance

## Cons
- Complex multi-layer stack (K8s + OpenFaaS + Kata + Firecracker)
- Cold start latency worse than warm containers
- High operational overhead for "serverless"
- Not as battle-tested as AWS Lambda