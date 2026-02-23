[← Back to Index](../index.md)

# DEPLOYMENT 57 — Bare Metal → Eucalyptus → KVM/Xen → VMs

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                      VMs                       │
├──────────────────────────────────────────────────┤
│                    KVM/Xen                     │
├──────────────────────────────────────────────────┤
│                   Eucalyptus                   │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              AWS-COMPATIBLE PRIVATE CLOUD                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  AWS CLI / Boto3 SDK (works against Eucalyptus EC2 API)  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  EC2-compatible API
┌────────────────────────────▼────────────────────────────────────┐
│              EUCALYPTUS CLOUD CONTROLLER                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CLC (Cloud Controller) │ Walrus (S3-compat storage)     │   │
│  │  CC (Cluster Controller) │ SC (Storage Controller)       │   │
│  │  NC (Node Controller) on each hypervisor node            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KVM / XEN HYPERVISOR NODES                         │
│  VMs (launched by NC agent via libvirt/KVM or Xen)             │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Historical: AWS-compatible private cloud. **Status: Effectively unmaintained — do not use for new deployments.**