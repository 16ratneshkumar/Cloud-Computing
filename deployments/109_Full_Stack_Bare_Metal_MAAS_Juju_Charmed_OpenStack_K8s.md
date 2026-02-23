[← Back to Index](../index.md)

# DEPLOYMENT 109 — Full Stack: Bare Metal → MAAS → Juju → Charmed OpenStack + K8s

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│            Charmed OpenStack + K8s             │
├──────────────────────────────────────────────────┤
│                      Juju                      │
├──────────────────────────────────────────────────┤
│                      MAAS                      │
├──────────────────────────────────────────────────┤
│             Full Stack: Bare Metal             │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              CLOUD WORKLOADS                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Tenant VMs (OpenStack Nova KVM VMs)                     │   │
│  │  Charmed Kubernetes clusters (Juju-deployed K8s)         │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              CHARMED OPENSTACK (Juju-deployed)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Juju Charms (software operators for each service)       │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  nova-cloud-controller charm                       │  │   │
│  │  │  nova-compute charm (on each compute node)         │  │   │
│  │  │  neutron-api charm (Neutron controller)            │  │   │
│  │  │  keystone charm │ cinder charm │ glance charm       │  │   │
│  │  │  ceph-mon charm │ ceph-osd charm                   │  │   │
│  │  │  percona-cluster charm (MariaDB Galera)            │  │   │
│  │  │  rabbitmq-server charm                             │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Juju controller manages charm lifecycle + relations     │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Juju deploys charms via SSH to MAAS nodes
┌────────────────────────────▼────────────────────────────────────┐
│              JUJU (service lifecycle manager)                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Juju Controller (manages all Juju operations)           │   │
│  │  juju deploy charmed-openstack-bundle                    │   │
│  │  juju add-unit nova-compute -n 5 (add 5 compute nodes)  │   │
│  │  juju upgrade-charm nova-compute (rolling upgrade)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Juju provisions via MAAS
┌────────────────────────────▼────────────────────────────────────┐
│              MAAS (Metal-as-a-Service)                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MAAS Region Controller (API + DB)                       │   │
│  │  MAAS Rack Controller (DHCP + TFTP per rack)             │   │
│  │  → Commissions nodes (discovers hardware)                │   │
│  │  → Deploys Ubuntu on nodes requested by Juju             │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  IPMI/Redfish power control + PXE
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL NODE POOL                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  All hardware managed by MAAS:                           │   │
│  │  Controller nodes (3) │ Compute nodes (N) │ OSD nodes (N)│   │
│  │  Network nodes (2)    │ Storage nodes     │               │   │
│  │  BMC (iDRAC/iLO/IPMI/Redfish) on all nodes               │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

DEPLOYMENT FLOW:
1. MAAS discovers hardware (PXE commission)
2. Juju adds MAAS as cloud
3. juju deploy charmed-openstack-bundle
4. Juju allocates MAAS nodes, deploys Ubuntu, runs charms
5. Charms configure OpenStack services and relations between them
6. OpenStack ready in hours (vs days manually)
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- MAAS discovers and commissions all bare metal hardware
- Juju uses MAAS as its cloud provider — requests machines, deploys Ubuntu, runs charm agents
- Charmed OpenStack bundle deploys ~20 charms that configure and relate all OpenStack services
- Juju relations wire services together (e.g., nova-compute → rabbitmq-server charm relation)
- Upgrades: `juju upgrade-charm` performs rolling upgrades with zero downtime

## Use Case
Canonical's reference architecture for OpenStack. Banks, telecoms, and research networks wanting a vendor-supported, automated OpenStack deployment on bare metal.

## Pros
- Fully automated deployment (hours, not weeks)
- Juju handles rolling upgrades automatically
- MAAS handles hardware lifecycle
- Canonical commercial support
- Full Ubuntu/Canonical stack

## Cons
- Ubuntu/Canonical ecosystem only
- Juju learning curve
- Charm quality varies by community charm
- Less common outside Canonical ecosystem