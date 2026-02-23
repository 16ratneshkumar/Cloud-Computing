[← Back to Index](../index.md)

# DEPLOYMENT 25 — Bare Metal → Nutanix AHV → VMs + Integrated Storage

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│            VMs + Integrated Storage            │
├──────────────────────────────────────────────────┤
│                  Nutanix AHV                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                  MANAGEMENT PLANE                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Prism Central (multi-cluster management)                │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  VM Mgmt │ Storage Mgmt │ Networking │ Flow (SDN)   │ │   │
│  │  │  Leap (DR) │ X-Play (automation) │ Calm (IaC)      │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  manages
┌────────────────────────────▼────────────────────────────────────┐
│              NUTANIX NODE (each node has AHV + CVM)             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           AHV HYPERVISOR (KVM-based)                     │   │
│  │   ┌───────────────────────────────────────────────────┐  │   │
│  │   │           GUEST VMs                               │  │   │
│  │   │  ┌──────────────┐  ┌──────────────┐              │  │   │
│  │   │  │  Windows VM  │  │  Linux VM    │              │  │   │
│  │   │  │  virtio-scsi │  │  virtio-net  │              │  │   │
│  │   │  └──────────────┘  └──────────────┘              │  │   │
│  │   └───────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │     CVM (Controller VM) — per node storage controller    │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  Stargate (I/O path) │ Curator (distributed ops)│    │   │
│  │  │  Cassandra (metadata ring) │ Zookeeper (coord)  │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  distributed storage fabric
┌────────────────────────────▼────────────────────────────────────┐
│         NUTANIX DISTRIBUTED STORAGE FABRIC (across nodes)       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1 SSD/HDD ──── Node 2 SSD/HDD ──── Node 3 SSD/HDD │   │
│  │           Replication Factor 2 or 3                      │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  Nutanix-Certified Nodes (NX / Dell XC / HPE DX / Lenovo HX)   │
│  CPU (Intel/AMD) │ RAM (ECC) │ NVMe SSD + HDD │ 10/25GbE NIC   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Each Nutanix node runs AHV (KVM-based) for VM compute and a CVM (Controller VM) for storage
- The CVM implements Nutanix's distributed storage, presenting a local virtual disk to guest VMs
- Data is replicated across CVM nodes via the Stargate I/O path
- Prism Central manages multiple clusters from a single pane
- Nutanix Flow provides micro-segmentation networking

## Use Case
Enterprise HCI replacing VMware + SAN. VDI, database hosting, and distributed enterprise workloads.

## Pros
- Integrated compute + storage — no external SAN
- Prism Central is excellent management UI
- Self-healing distributed storage
- Hardware-agnostic

## Cons
- Expensive licensing
- AHV less feature-rich than ESXi historically
- Nutanix software lock-in