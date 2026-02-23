[← Back to Index](../index.md)

# DEPLOYMENT 47 — Bare Metal → OpenStack Ironic → Bare Metal Nodes

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                Bare Metal Nodes                │
├──────────────────────────────────────────────────┤
│                OpenStack Ironic                │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT / OPERATOR INTERFACES                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  openstack baremetal node list                           │   │
│  │  openstack baremetal node deploy <node>                  │   │
│  │  Nova (uses Ironic as compute driver)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENSTACK IRONIC                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ironic-api (REST API for bare metal mgmt)               │   │
│  │  ironic-conductor (performs actual provisioning actions) │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  PROVISIONING WORKFLOW:                            │  │   │
│  │  │  1. Power on node via IPMI/Redfish/iDRAC           │  │   │
│  │  │  2. Node PXE boots → Ironic Python Agent (IPA)     │  │   │
│  │  │  3. IPA reports hardware details back to Ironic    │  │   │
│  │  │  4. Ironic writes OS image to node disk (via IPA)  │  │   │
│  │  │  5. Node reboots into new OS                       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  DRIVERS:                                                │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  IPMI driver   │  Redfish driver   │  iDRAC driver │  │   │
│  │  │  (power/boot   │  (modern BMC API) │  (Dell DRAC)  │  │   │
│  │  │  control)      │                   │               │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  DEPLOY INTERFACES:                                      │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  direct (IPA writes image) │ iscsi │ ramdisk       │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  IPMI / Redfish to BMC
┌────────────────────────────▼────────────────────────────────────┐
│              BARE METAL NODES (target hardware)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Node 1                  Node 2               Node 3    │   │
│  │  ┌───────────────────┐   ┌──────────────┐   ┌────────┐  │   │
│  │  │  BMC (iDRAC/iLO/  │   │  BMC         │   │  BMC   │  │   │
│  │  │  IPMI/Redfish)    │   │              │   │        │  │   │
│  │  │  ← Ironic controls│   │              │   │        │  │   │
│  │  │    power + boot   │   │              │   │        │  │   │
│  │  └───────────────────┘   └──────────────┘   └────────┘  │   │
│  │  ┌───────────────────────────────────────────────────┐   │   │
│  │  │  PXE Network Boot (DHCP → TFTP → initrd/kernel)  │   │   │
│  │  │  → boots Ironic Python Agent (ramdisk)            │   │   │
│  │  │  → IPA calls home to Ironic API                   │   │   │
│  │  │  → Ironic deploys OS image to disk                │   │   │
│  │  │  → Node reboots into full OS (RHEL/Ubuntu/etc.)   │   │   │
│  │  └───────────────────────────────────────────────────┘   │   │
│  │                  BARE METAL (raw hardware)                │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- Ironic stores bare metal node inventory (MAC addresses, IPMI creds, specs)
- When provisioning is requested, Ironic uses IPMI/Redfish to power on the node and set PXE boot
- Node PXE boots a minimal ramdisk containing Ironic Python Agent (IPA)
- IPA calls back to Ironic API reporting hardware, then Ironic instructs IPA to write the OS image to disk
- Node reboots into the fully provisioned OS — no hypervisor involved

## Use Case
Cloud API for bare metal servers — giving tenants raw hardware via the same OpenStack APIs as VMs. Used for HPC workloads, databases, and GPU workloads that can't tolerate VM overhead.

## Pros
- Cloud API for hardware provisioning
- Automated PXE/BMC provisioning
- No virtualization overhead for the workload
- MAAS-like bare metal capabilities within OpenStack

## Cons
- Slow provisioning (minutes per node)
- Complex BMC management
- Tenant isolation harder (physical hardware sharing)
- Node recovery requires full re-imaging