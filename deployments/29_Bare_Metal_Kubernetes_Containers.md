[← Back to Index](../index.md)

# DEPLOYMENT 29 — Bare Metal → Kubernetes → Containers

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Containers                   │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES CLIENTS                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  kubectl     │  │  Helm        │  │  ArgoCD / FluxCD     │  │
│  │  (CLI)       │  │  (package    │  │  (GitOps operator)   │  │
│  │              │  │   manager)   │  │                      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │  REST / HTTPS (port 6443)
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES CONTROL PLANE                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kube-apiserver — REST API, auth, admission control      │   │
│  │  etcd — distributed key-value store (cluster state)      │   │
│  │  kube-scheduler — assigns pods to nodes                  │   │
│  │  kube-controller-manager — reconciliation loops          │   │
│  │  cloud-controller-manager — optional, cloud integration  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  kubelet on each node
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES WORKER NODES                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1                   │  Node 2         │  Node 3    │   │
│  │  ┌─────────────────────┐  │  ┌───────────┐  │  ┌──────┐  │   │
│  │  │  kubelet (node agent)│  │  │  kubelet  │  │  │kubelt│  │   │
│  │  │  kube-proxy (netw)   │  │  │  kube-    │  │  │kproxy│  │   │
│  │  └─────────────────────┘  │  │  proxy    │  │  └──────┘  │   │
│  │  ┌─────────────────────┐  │  └───────────┘  │            │   │
│  │  │  PODS               │  │                 │            │   │
│  │  │  ┌──────┐  ┌──────┐ │  │  ┌──────────┐  │  ┌──────┐  │   │
│  │  │  │Pod A │  │Pod B │ │  │  │  Pod C   │  │  │ Pod D│  │   │
│  │  │  │(nginx│  │(app) │ │  │  │  (mysql) │  │  │(cache│  │   │
│  │  │  │cont) │  │      │ │  │  │          │  │  │ cont)│  │   │
│  │  │  └──────┘  └──────┘ │  │  └──────────┘  │  └──────┘  │   │
│  │  └─────────────────────┘  │                │            │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ADD-ONS (DaemonSets / System Pods)                      │   │
│  │  ┌────────────┐  ┌──────────┐  ┌─────────┐  ┌────────┐  │   │
│  │  │  CNI plugin│  │  CSI     │  │  CoreDNS│  │ metrics│  │   │
│  │  │  (Calico / │  │  driver  │  │  (DNS)  │  │ server │  │   │
│  │  │   Cilium / │  │  (storage│  │         │  │        │  │   │
│  │  │   Flannel) │  │   plugin)│  │         │  │        │  │   │
│  │  └────────────┘  └──────────┘  └─────────┘  └────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  CRI (containerd / CRI-O)
┌────────────────────────────▼────────────────────────────────────┐
│              CONTAINER RUNTIME (per node)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  containerd (or CRI-O)                                   │   │
│  │  → pulls images from registry                            │   │
│  │  → creates container namespaces                          │   │
│  │  → manages container lifecycle                           │   │
│  │  runc / crun — OCI container runtime (actual container)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (each node)                     │
│  cgroups v2 │ namespaces │ iptables/eBPF │ overlay filesystem   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Node 1          │  Node 2          │  Node 3                  │
│  CPU│RAM│NVMe    │  CPU│RAM│NVMe    │  CPU│RAM│NVMe            │
│  25GbE NIC       │  25GbE NIC       │  25GbE NIC               │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- kube-apiserver is the central REST API; all components talk to it
- etcd stores all cluster state (only control plane writes to it)
- Scheduler watches for unscheduled pods and assigns them to nodes based on resource requests and affinity rules
- kubelet on each node communicates with apiserver and manages pods via containerd
- CNI plugin handles pod networking (assigns IPs, routing between pods across nodes)
- CSI driver handles persistent storage (provisioning PVCs on NFS/Ceph/local)

## Use Case
Cloud-native microservices on bare metal for maximum performance. Financial services, telecoms, and tech companies wanting no VM overhead.

## Pros
- No VM overhead — maximum density
- Declarative GitOps management
- Massive ecosystem (Helm, operators, CNCF tools)
- Auto-scaling, self-healing
- Multi-cloud portable workload definitions

## Cons
- No native VM support (need KubeVirt)
- Bare metal provisioning complex (kubeadm / k3s / Talos)
- etcd requires careful backup and maintenance
- Steep learning curve
- Networking complexity (CNI plugins)