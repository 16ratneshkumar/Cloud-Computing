[← Back to Index](../index.md)

# DEPLOYMENT 76 — Bare Metal → KVM → VM (Host) → KVM → Nested VMs (Guest)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│             Workload (App/Process)             │
├──────────────────────────────────────────────────┤
│               Nested VMs (Guest)               │
├──────────────────────────────────────────────────┤
│                      KVM                       │
├──────────────────────────────────────────────────┤
│                   VM (Host)                    │
├──────────────────────────────────────────────────┤
│                      KVM                       │
├──────────────────────────────────────────────────┤
│                   Bare Metal                   │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              L2 NESTED VM WORKLOADS                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  L2 Nested VM A         L2 Nested VM B                  │   │
│  │  (Ubuntu 22.04)         (Windows 11 — test)             │   │
│  │  vCPU, RAM, disk        vCPU, RAM, disk                 │   │
│  │  (inside L1 VM)         (inside L1 VM)                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  L2 hypervisor (inside L1 VM)
┌────────────────────────────▼────────────────────────────────────┐
│              L1 VIRTUAL MACHINE (Guest OS + KVM)                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Guest OS: Ubuntu 22.04 / RHEL 9                         │   │
│  │  KVM enabled inside guest (nested KVM)                   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  L1 KVM module (kvm.ko inside guest)               │  │   │
│  │  │  QEMU processes for L2 VMs                         │  │   │
│  │  │  libvirt (optional, for L2 VM management)          │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Exposes /dev/kvm to L2 QEMU processes                   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  (L1 VM has nested=on CPU feature exposed by L0 host)          │
└────────────────────────────┬────────────────────────────────────┘
                             │  L0 → L1 VM (shadow EPT tables)
┌────────────────────────────▼────────────────────────────────────┐
│              L0 HOST HYPERVISOR (bare metal)                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  KVM + QEMU (L0 level)                                   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  L1 VM configured with:                            │  │   │
│  │  │  cpu: host-passthrough (or -cpu kvm64,vmx=on)      │  │   │
│  │  │  → exposes VMX/SVM instructions to guest           │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Shadow EPT: L0 maps L2 guest memory directly           │   │
│  │  (KVM handles nested EPT page walk — performance cost)   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              L0 LINUX KERNEL + KVM MODULE (host)                │
│  kvm.ko │ kvm-intel.ko (nested=1 module param)                  │
│  OR kvm-amd.ko (nested=1)                                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL                                │
│  CPU: Intel VT-x + EPT (nested EPT) or AMD-V + RVI             │
│  RAM: Large pool (L0 + L1 + L2 overhead)                        │
└─────────────────────────────────────────────────────────────────┘

PERFORMANCE OVERHEAD:
┌─────────────────────────────────────────────────────────────────┐
│  L0 (bare metal KVM):    ~2-5% overhead vs native              │
│  L1 (VM in KVM):         ~10-15% overhead vs L0                │
│  L2 (nested VM):         ~40-70% overhead vs L0 (total)        │
│  Memory: 3 EPT page tables maintained simultaneously            │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Isolation:** Hardware-assisted virtualization (VT-x/AMD-V).
- **Storage:** Virtual disks (qcow2/vmdk) on shared or local storage.
- **Management:** Orchestrated via native CLI or high-level API controllers.


## How It Works
- L0 (bare metal) KVM host enables nested virtualization: `echo 1 > /sys/module/kvm_intel/parameters/nested`
- L1 VM is created with `cpu=host-passthrough` or `vmx=on` — VMX instructions exposed to guest
- Inside L1, the guest Linux kernel loads kvm.ko and sees /dev/kvm
- L2 VMs are created inside L1 using QEMU
- L0 KVM intercepts L1's VMX instructions and emulates them using shadow EPT tables

## Use Case
Cloud providers offering "bare metal-like" VM instances with VT-x passthrough, CI/CD for hypervisor testing (testing KVM inside a CI VM), and security research.

## Pros
- Test hypervisor environments without physical hardware
- Cloud providers can expose KVM to tenants
- Useful for Kubernetes-in-VM testing
- Vagrant and cloud CI/CD uses this

## Cons
- Significant performance penalty (40-70% at L2)
- Complex memory management (shadow EPT)
- Higher VM exit rate at L2
- Not suitable for production workloads