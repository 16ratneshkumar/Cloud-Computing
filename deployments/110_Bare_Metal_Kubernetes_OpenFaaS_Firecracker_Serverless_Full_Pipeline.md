[← Back to Index](../index.md)

# DEPLOYMENT 110 — Bare Metal → Kubernetes → OpenFaaS → Firecracker → Serverless (Full Pipeline)

## Deployment Stack Overlay

```
┌──────────────────────────────────────────────────┐
│           Serverless Function (Code)             │
├──────────────────────────────────────────────────┤
│          Firecracker MicroVM Sandbox             │
├──────────────────────────────────────────────────┤
│             OpenFaaS Runtime Layer               │
├──────────────────────────────────────────────────┤
│            Kubernetes Pod / Runtime              │
├──────────────────────────────────────────────────┤
│          Host OS / KVM Kernel Module             │
├──────────────────────────────────────────────────┤
│               Bare Metal Hardware                │
└──────────────────────────────────────────────────┘
```

## Detailed System Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│              DEVELOPER / CLIENT LAYER                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Function developer (faas-cli deploy --gateway https://..│   │
│  │  End user client (HTTP POST /function/myFunction)         │   │
│  │  CI/CD pipeline (auto-deploy on git push)                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              OPENFAAS PLATFORM LAYER                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenFaaS Gateway (K8s Deployment)                       │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Reverse proxy → routes to function replicas       │  │   │
│  │  │  Scale-to-zero: scales function pods to 0 when idle│  │   │
│  │  │  Scale-from-zero: starts pods on first request     │  │   │
│  │  │  Authentication (basic/JWT)                        │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Queue-worker (NATS Streaming)                           │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  Async invocations → NATS topic → worker queues    │  │   │
│  │  │  Retry logic │ Dead letter queue                    │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │  Prometheus (metrics) → KEDA (event-driven autoscaling) │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  function pods with RuntimeClass
┌────────────────────────────▼────────────────────────────────────┐
│              KUBERNETES SCHEDULING + RUNTIME LAYER              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Function A Pod         Function B Pod       Function C  │   │
│  │  RuntimeClass:          RuntimeClass:         RuntimeCls: │   │
│  │  kata-firecracker       kata-qemu (fallback)  runc       │   │
│  │  ┌──────────────────┐   ┌────────────────┐  ┌─────────┐ │   │
│  │  │Firecracker MicroVM│  │QEMU MicroVM    │  │Container│ │   │
│  │  │Own Linux kernel   │  │(device compat) │  │(shared  │ │   │
│  │  │~125ms cold start  │  │~300ms cold     │  │ kernel) │ │   │
│  │  │<5MB VMM overhead  │  │~30MB overhead  │  │<1MB     │ │   │
│  │  └──────────────────┘   └────────────────┘  └─────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  Kata runtime creates VMs
┌────────────────────────────▼────────────────────────────────────┐
│              KATA CONTAINERS RUNTIME LAYER                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kata-runtime (containerd shim v2)                       │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  kata-fc backend → launches Firecracker VMM        │  │   │
│  │  │  kata-qemu backend → launches QEMU (fallback)      │  │   │
│  │  │  Kata agent (inside each VM) → OCI compliance      │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              FIRECRACKER + JAILER LAYER                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Per-function MicroVM:                                   │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │  jailer process (cgroup + seccomp + chroot sandbox) │  │   │
│  │  │  └── firecracker VMM (Rust, ~1MB binary)           │  │   │
│  │  │       └── MicroVM:                                  │  │   │
│  │  │            Minimal Linux kernel (~5.10 LTS)         │  │   │
│  │  │            Shared read-only rootfs (overlay)        │  │   │
│  │  │            Per-VM writable layer (overlayfs)        │  │   │
│  │  │            virtio-net (TAP device)                  │  │   │
│  │  │            virtio-blk (function code disk)          │  │   │
│  │  │            Kata agent (receives container cmds)     │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  /dev/kvm ioctls
┌────────────────────────────▼────────────────────────────────────┐
│              KVM KERNEL MODULE                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kvm.ko + kvm-intel.ko / kvm-amd.ko                      │   │
│  │  EPT (Extended Page Tables) — hardware memory isolation  │   │
│  │  VMX/SVM — hardware CPU virtualization                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              LINUX KERNEL (HOST OS)                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  TUN/TAP devices (one per MicroVM for virtio-net)        │   │
│  │  cgroups v2 (resource limits per jailer/MicroVM)         │   │
│  │  seccomp (syscall filter for Firecracker process)        │   │
│  │  overlayfs (shared rootfs + per-VM writable layer)       │   │
│  │  iptables/eBPF (pod network rules via CNI plugin)        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│              STORAGE + NETWORK LAYER                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Function code: container registry → init container      │   │
│  │                 → mounts as virtio-blk in MicroVM        │   │
│  │  Shared rootfs:  cached on each node (NVMe)              │   │
│  │  Pod networking: Calico/Cilium CNI → eBPF routing        │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                       BARE METAL (Host Servers)                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  CPU: Intel VT-x or AMD-V (mandatory for KVM)           │   │
│  │  Cores: 64+ (many concurrent MicroVMs need CPU headroom)│   │
│  │  RAM: 256GB+ (1000 MicroVMs × 128MB avg = 128GB)        │   │
│  │  NVMe: Fast local (rootfs snapshot caching)             │   │
│  │  NIC: 25GbE+ (TAP device I/O for many MicroVMs)         │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Technical Implementation Details
- **Networking:** `Nginx Ingress` routes traffic to `OpenFaaS Gateway`. Pods use `Kata Containers` with `TAP` devices to bridge into the Firecracker MicroVM.
- **Storage:** Function code is pulled as a container image and mounted as a read-only `virtio-blk` device in the MicroVM.
- **Security:** Hardware-level isolation via `KVM` (VMX/EPT). Each function runs in its own specialized `MicroVM` sandbox.
- **Management:** Scaled-to-zero by `OpenFaaS`. Orchestrated by `Kubernetes` using a custom `RuntimeClass`.

## How It Works
- OpenFaaS Gateway receives all function HTTP requests and routes them to appropriate pods
- Scale-to-zero: when no traffic, OpenFaaS scales function pods to 0 replicas
- On first request (cold start): K8s schedules new pod with kata-firecracker RuntimeClass
- containerd calls kata-runtime shim which starts a Firecracker VMM
- The jailer sandboxes Firecracker (cgroup + seccomp + chroot)
- Firecracker boots a minimal Linux kernel in ~125ms
- Kata agent inside the VM receives the OCI container start command
- Function process starts inside the VM and processes the request

## Use Case
Self-hosted serverless platform for organizations requiring strong security isolation per function invocation (multi-tenant SaaS, security-sensitive compute).

## Pros
- KVM hardware isolation per function
- ~125ms cold start
- Full control over hardware and data

## Cons
- Extremely complex multi-layer stack
- Cold start slower than pure containers
- KVM hardware support required