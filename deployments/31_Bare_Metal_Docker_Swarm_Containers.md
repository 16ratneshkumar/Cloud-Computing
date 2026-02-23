[← Back to Index](../index.md)

# DEPLOYMENT 31 — Bare Metal → Docker Swarm → Containers

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Containers                   │
├──────────────────────────────────────────────────┤
│                  Docker Swarm                  │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                    MANAGEMENT CLIENTS                           │
│  ┌────────────────────┐  ┌──────────────────────────────────┐  │
│  │  docker CLI        │  │  docker stack deploy             │  │
│  │  (docker service   │  │  (Compose file → Swarm stack)    │  │
│  │   ls/scale/update) │  │                                  │  │
│  └────────────────────┘  └──────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │  Docker API (port 2377 swarm / 2376 TLS)
┌────────────────────────────▼────────────────────────────────────┐
│              DOCKER SWARM CLUSTER                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MANAGER NODES  (odd count: 1, 3, or 5 for HA)           │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Manager 1 (Leader)   Manager 2     Manager 3    │    │   │
│  │  │  ┌──────────────┐                               │    │   │
│  │  │  │  Raft        │  ← consensus for cluster state│    │   │
│  │  │  │  consensus   │                               │    │   │
│  │  │  │  (leader     │                               │    │   │
│  │  │  │   election)  │                               │    │   │
│  │  │  └──────────────┘                               │    │   │
│  │  │  Scheduler (places services on workers)         │    │   │
│  │  │  Dispatcher (sends tasks to workers)            │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  WORKER NODES                                            │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  Worker 1         Worker 2          Worker 3        │ │   │
│  │  │  ┌────────────┐   ┌────────────┐   ┌────────────┐  │ │   │
│  │  │  │ Service A  │   │ Service A  │   │ Service B  │  │ │   │
│  │  │  │ replica 1  │   │ replica 2  │   │ replica 1  │  │ │   │
│  │  │  │ (container)│   │ (container)│   │ (container)│  │ │   │
│  │  │  └────────────┘   └────────────┘   └────────────┘  │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OVERLAY NETWORK (ingress + custom)                      │   │
│  │  VXLAN-based overlay │ Routing mesh (any node → service) │   │
│  │  Encrypted overlay traffic (optional)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              DOCKER ENGINE (each node)                          │
│  dockerd │ containerd │ runc │ CNI bridge │ iptables NAT        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    LINUX KERNEL (each node)                     │
│  cgroups │ namespaces │ VXLAN │ iptables │ overlayfs            │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Node 1 (mgr+wkr)  │  Node 2 (worker)  │  Node 3 (worker)     │
│  CPU│RAM│SSD        │  CPU│RAM│SSD       │  CPU│RAM│SSD         │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Swarm mode is built into Docker Engine — `docker swarm init` on one node creates the cluster
- Manager nodes use Raft consensus to maintain cluster state
- Services are defined (replicas, image, ports) and the scheduler places tasks (containers) on workers
- Overlay network uses VXLAN to tunnel container traffic across nodes
- Routing mesh: any node forwards traffic to the correct service replica regardless of where it runs

## Use Case
Small to medium container orchestration for teams already using Docker. Internal apps, simple microservices, and migration step from Docker Compose to multi-node.

## Pros
- Zero extra software — built into Docker Engine
- Docker Compose-like stack files (familiar syntax)
- Simple to set up and understand
- Good for small teams not ready for Kubernetes

## Cons
- Docker Inc has deprioritized Swarm development
- No Helm/operator ecosystem
- Limited RBAC, no custom resource types
- Not recommended for new large-scale production deployments
- Kubernetes has effectively won the orchestration market