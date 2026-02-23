[← Back to Index](../index.md)

# DEPLOYMENT 51 — Bare Metal → MAAS → OpenStack / Kubernetes / KVM Hosts

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│       OpenStack / Kubernetes / KVM Hosts       │
├──────────────────────────────────────────────────┤
│                      MAAS                      │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              OPERATOR / AUTOMATION CLIENTS                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  maas CLI (maas admin machines)                          │   │
│  │  MAAS Web UI (port 5240)                                 │   │
│  │  Juju (service deployment on MAAS-managed nodes)         │   │
│  │  Terraform MAAS provider                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  MAAS API (REST)
┌────────────────────────────▼────────────────────────────────────┐
│              MAAS (Metal-as-a-Service) PLATFORM                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MAAS Region Controller                                  │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  API server  │  PostgreSQL DB  │  DNS (bind9)      │  │   │
│  │  │  Machine state machine (NEW→COMMISSIONING→READY    │  │   │
│  │  │  →ALLOCATED→DEPLOYING→DEPLOYED)                    │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  MAAS Rack Controller (per rack / L2 broadcast domain)   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  DHCP server │ TFTP server │ HTTP image cache      │  │   │
│  │  │  BMC relay   │ proxy DHCP  │                       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  COMMISSIONING FLOW:                                     │   │
│  │  1. Node powers on → PXE → commissioning OS (Ubuntu)    │   │
│  │  2. MAAS agent collects: CPU, RAM, disk, NIC details     │   │
│  │  3. Node enters READY state in MAAS inventory            │   │
│  │  DEPLOYMENT FLOW:                                        │   │
│  │  4. Operator allocates + deploys node (target OS)        │   │
│  │  5. Node PXE boots deployer image                        │   │
│  │  6. Target OS written to disk via curtin                 │   │
│  │  7. Node reboots into target OS (Ubuntu/RHEL/etc.)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  IPMI/Redfish power control + PXE
┌────────────────────────────▼────────────────────────────────────┐
│              BARE METAL NODE POOL                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  DEPLOYED AS: OpenStack Controller/Compute/Storage Nodes │   │
│  │            OR: Kubernetes Control Plane / Worker Nodes   │   │
│  │            OR: Standalone KVM Hypervisor Hosts           │   │
│  │            OR: Any Ubuntu / RHEL / Custom workload       │   │
│  │  All managed via BMC (iDRAC/iLO/IPMI/Redfish)           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- MAAS discovers nodes via PXE / BMC and commissions them (collects hardware inventory)
- Operator allocates nodes to workloads (OpenStack, K8s, KVM) and deploys target OS
- MAAS handles PXE boot, DNS, DHCP for all nodes in the physical network
- Juju can deploy complex services (like Charmed OpenStack) on MAAS-provisioned nodes
- All subsequent infrastructure (OpenStack, K8s) runs on nodes MAAS has provisioned

## Use Case
Ubuntu/Canonical ecosystem bare metal automation. Foundation for Charmed OpenStack and Charmed Kubernetes deployments.

## Pros
- Full hardware lifecycle management
- API-driven node provisioning
- IPMI/Redfish/iDRAC/iLO support
- Integrates with Juju for service deployment

## Cons
- Ubuntu/Canonical focused
- Requires MAAS region/rack controllers
- Less flexible for non-Ubuntu stacks