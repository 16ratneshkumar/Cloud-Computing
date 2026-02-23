[← Back to Index](../index.md)

# DEPLOYMENT 101 — IBM Z Mainframe → z/VM → Linux → K8s → Containers

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│                   Containers                   │
├──────────────────────────────────────────────────┤
│                      K8s                       │
├──────────────────────────────────────────────────┤
│                     Linux                      │
├──────────────────────────────────────────────────┤
│                      z/VM                      │
├──────────────────────────────────────────────────┤
│                IBM Z Mainframe                 │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              KUBERNETES WORKLOADS (on mainframe)                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kubernetes pods (Java, Node.js, Python apps)            │   │
│  │  Mainframe-native workloads (COBOL, z/OS services)       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES (on Linux on Z)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Red Hat OpenShift on IBM Z (S390X architecture)         │   │
│  │  OR upstream K8s on Ubuntu/RHEL on Z                    │   │
│  │  containerd │ CRI-O │ s390x container images            │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  K8s runs inside Linux on z/VM guest
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX ON Z (z/VM guests)                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RHEL for Z (s390x) │ Ubuntu on Z                        │   │
│  │  Multiple Linux guests on same z/VM                      │   │
│  │  K8s control plane VMs + K8s worker VMs                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              z/VM HYPERVISOR                                    │
│  CP (Control Program) │ CMS │ thousands of Linux guests        │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              IBM Z HARDWARE (z16 / LinuxONE III)                │
│  S390X CPUs │ Pervasive Encryption │ Up to 32TB RAM            │
│  DASD / Flash Express storage │ Redundant everything           │
│                  BARE METAL (IBM Z)                             │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** Uses CNI plugins (Calico/Cilium) for pod-to-pod communication.
- **Storage:** Persistent Volumes (PV) managed via CSI drivers.
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## Use Case
Banks and insurance companies running Kubernetes workloads directly on mainframe — combining cloud-native container applications with COBOL batch processing on the same hardware.

## Pros
- K8s on mainframe hardware — 6-nines availability
- Pervasive encryption for all container data
- Same hardware as core banking systems
- Massive consolidation — thousands of Linux VMs + K8s on one machine

## Cons
- IBM Z licensing and hardware costs are extreme
- S390X architecture limits container image availability
- Very specialized expertise