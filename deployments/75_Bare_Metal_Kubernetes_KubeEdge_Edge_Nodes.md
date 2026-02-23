[← Back to Index](../index.md)

# DEPLOYMENT 75 — Bare Metal → Kubernetes → KubeEdge → Edge Nodes

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Edge Nodes                   │
├──────────────────────────────────────────────────┤
│                    KubeEdge                    │
├──────────────────────────────────────────────────┤
│                   Kubernetes                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              CLOUD MANAGEMENT LAYER                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kubectl (manages both cloud and edge nodes uniformly)   │   │
│  │  Cloud K8s cluster (control plane for all edge nodes)    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  WebSocket tunnel (HTTPS)
┌────────────────────────────▼────────────────────────────────────┐
│              KUBEEDGE CLOUD SIDE (runs in cloud K8s)            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CloudCore (KubeEdge cloud component — K8s pod)          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  EdgeHub — manages WebSocket connections to edges  │  │   │
│  │  │  EdgeController — syncs K8s resources to edges     │  │   │
│  │  │  DeviceController — IoT device twin management     │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  WebSocket (works through NAT/firewall)
┌────────────────────────────▼────────────────────────────────────┐
│              EDGE NODE (remote location — may be offline)       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  EdgeCore (KubeEdge edge component)                      │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Edged — local kubelet-like pod manager            │  │   │
│  │  │  EdgeHub — WebSocket client to CloudCore           │  │   │
│  │  │  EventBus — MQTT broker for IoT device events      │  │   │
│  │  │  DeviceTwin — local digital twin for IoT devices   │  │   │
│  │  │  MetaManager — local metadata cache (works offline)│  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Edge Pods (containers on edge node)               │  │   │
│  │  │  ┌──────────────────────────────────────────────┐  │  │   │
│  │  │  │  AI/ML inference (TensorFlow Lite pod)       │  │  │   │
│  │  │  │  IoT data aggregator pod                     │  │  │   │
│  │  │  │  Protocol adapter pod (Modbus/OPC-UA bridge) │  │  │   │
│  │  │  └──────────────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  IoT DEVICES (connected to edge node)              │  │   │
│  │  │  Sensors │ Cameras │ PLCs │ Industrial equipment   │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Edge Hardware)                │
│  Raspberry Pi / Jetson Nano / Industrial PC / ARM SBC           │
│  CPU (ARM64 or x86) │ RAM 2GB+ │ Flash/eMMC │ WiFi/LTE/Eth     │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- CloudCore runs in the central cloud K8s cluster and manages all edge nodes
- EdgeCore runs on each edge device and registers with CloudCore via WebSocket
- EdgeCore's MetaManager caches pod specs and device configs locally — pods continue running even if cloud is unreachable
- DeviceTwin provides a digital twin for each IoT device (reported state + desired state)
- EventBus bridges MQTT from IoT devices into K8s events visible at the cloud level
- kubectl from the cloud can list and manage pods on remote edge nodes

## Use Case
Industrial IoT, smart manufacturing, autonomous vehicles, smart cities — where Kubernetes-managed containers need to run at remote locations with unreliable connectivity.

## Pros
- Works offline — edge pods survive cloud disconnection
- Kubernetes API consistency from cloud to edge
- IoT device twin management built-in
- CNCF project with active community

## Cons
- WebSocket tunnel — limited throughput vs direct K8s
- Limited edge node resources vs cloud K8s features
- Complex debug across cloud-edge boundary
- MQTT bridging adds complexity for IoT