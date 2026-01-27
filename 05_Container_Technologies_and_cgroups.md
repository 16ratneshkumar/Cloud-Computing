# Container Technologies and cgroups
## Software-Level Virtualization in Linux

---

## Table of Contents
1. [What are cgroups?](#what-are-cgroups)
2. [cgroup Subsystems](#cgroup-subsystems)
3. [Container Architecture](#container-architecture)
4. [Docker Architecture](#docker-architecture)
5. [Kubernetes and cgroups](#kubernetes-and-cgroups)
6. [Practical Examples](#practical-examples)

---

## What are cgroups?

**Control Groups (cgroups)** are a Linux kernel feature that provides a mechanism for aggregating sets of processes and their children into hierarchical groups with specialized resource management behavior.

### Core Concept: Resource Namespace Isolation

Unlike hardware virtualization (KVM, Xen) which creates complete virtual machines, cgroups implement **OS-level virtualization** by partitioning system resources among process groups while sharing the same kernel.

```
Container = cgroups + namespaces + layered filesystem + process isolation

cgroups answer: "How much can you use?"
Namespaces answer: "What can you see?"
```

---

## cgroup Subsystems (Controllers)

### 1. CPU Controller

**cpu.shares** - Proportional CPU allocation (weight-based)
**cpu.cfs_quota_us / cpu.cfs_period_us** - Hard CPU limits

```bash
# Limit a process group to 50% of one CPU
echo 50000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_quota_us
echo 100000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_period_us

# CPU Bandwidth = (quota / period) * 100%
# 50000 / 100000 = 50%
```

**cpuset.cpus** - Pin processes to specific CPU cores
**cpuset.mems** - Bind to specific NUMA memory nodes

---

### 2. Memory Controller

```bash
# Hard memory limit (512MB)
echo 536870912 > /sys/fs/cgroup/memory/container/memory.limit_in_bytes

# Soft limit (reclaimed under pressure)
echo 268435456 > /sys/fs/cgroup/memory/container/memory.soft_limit_in_bytes
```

**Key Files:**
- `memory.limit_in_bytes` - Hard limit
- `memory.usage_in_bytes` - Current usage
- `memory.oom_control` - OOM killer behavior
- `memory.stat` - Detailed statistics

---

### 3. Block I/O Controller (blkio)

```bash
# Weight-based I/O allocation (100-1000)
echo 500 > /sys/fs/cgroup/blkio/mygroup/blkio.weight

# Throttle read throughput
echo "8:0 10485760" > /sys/fs/cgroup/blkio/mygroup/blkio.throttle.read_bps_device
# Device 8:0 limited to 10MB/s reads
```

---

### 4. Other Controllers

| Controller | Purpose |
|------------|---------|
| **net_cls** | Tag packets for traffic control |
| **net_prio** | Network interface priorities |
| **devices** | Device whitelist/blacklist |
| **freezer** | Suspend/resume processes atomically |
| **pids** | Limit process count (prevents fork bombs) |

---

## Hierarchical Organization

```
/sys/fs/cgroup/
├── cpu/
│   ├── system.slice/
│   ├── user.slice/
│   └── docker/
│       ├── container1/
│       └── container2/
├── memory/
│   └── docker/
│       ├── container1/
│       └── container2/
└── blkio/
    └── docker/
```

**Key Mechanisms:**
1. **Task Assignment**: Processes assigned via PID
2. **Hierarchical Inheritance**: Children inherit parent limits
3. **Resource Accounting**: Kernel tracks usage per cgroup
4. **Enforcement**: Scheduler and memory manager enforce limits

---

## cgroups v1 vs v2

| Aspect | cgroups v1 | cgroups v2 |
|--------|-----------|-----------|
| Hierarchy | Multiple (one per controller) | Single unified tree |
| Flexibility | More flexible, complex | Simpler, consistent |
| Granularity | Process level | Thread level support |
| Design | Legacy | Modern, container-focused |

**cgroups v2 Structure:**
```
/sys/fs/cgroup/
├── cgroup.controllers
├── cgroup.procs
├── system.slice/
│   ├── cpu.max
│   ├── memory.max
│   └── io.max
└── user.slice/
```

---

## Container Architecture

### What is a Container?

```
┌─────────────────────────────────────┐
│     Application Process             │
├─────────────────────────────────────┤
│     Container Runtime (containerd)  │
├─────────────────────────────────────┤
│     OCI Runtime (runc)              │
├─────────────────────────────────────┤
│     Linux Kernel Features:          │
│     • cgroups (resource limits)     │
│     • Namespaces (isolation)        │
│     • seccomp (syscall filtering)   │
│     • SELinux/AppArmor (MAC)        │
└─────────────────────────────────────┘
```

### Linux Namespaces Details

**PID Namespace:**
```
Host:      PID 1234 (container process)
Container: PID 1 (appears as init process)
```

**Network Namespace:**
```
Host:      eth0 (10.0.0.5)
Container: eth0 (172.17.0.2 - virtual interface)
```

**Mount Namespace:**
```
Host:      / (full filesystem)
Container: / (isolated filesystem tree)
```

**User Namespace:**
```
Container root (UID 0) → Host UID 100000
(Unprivileged on host, root inside container)
```

---

### Container Storage (Overlay Filesystem)

```
Container View:    Actually on Disk:
/                  lowerdir (read-only layers):
├── bin/           ├── layer1 (base OS)
├── etc/           ├── layer2 (dependencies)
└── var/           └── layer3 (application)
                   upperdir (read-write):
                   └── container changes
                   workdir (overlay working dir)

Final view = overlay of all layers
```

**Benefits:**
- Layers shared between containers
- Copy-on-write for modifications
- Efficient storage usage

---

## Docker Architecture

```
┌────────────────────────────────────────────┐
│  Docker Client (docker CLI)                │
└──────────────────┬─────────────────────────┘
                   │ REST API
┌──────────────────▼─────────────────────────┐
│  Docker Daemon (dockerd)                   │
│  ├─ Image Management                       │
│  ├─ Container Lifecycle                    │
│  ├─ Network Management                     │
│  └─ Volume Management                      │
├────────────────────────────────────────────┤
│  containerd                                │
│  ├─ Container Supervision                  │
│  ├─ Image Distribution                     │
│  └─ Snapshotter (storage)                  │
├────────────────────────────────────────────┤
│  runc (OCI runtime)                        │
│  ├─ Creates namespaces                     │
│  ├─ Sets up cgroups                        │
│  └─ Starts container process               │
└────────────────────────────────────────────┘
```

### Docker run Execution Flow

When you run `docker run -m 512m --cpus=1.5 nginx`:

**Step 1:** Docker CLI → Docker Daemon

**Step 2:** Daemon translates to OCI spec:
```json
{
  "linux": {
    "resources": {
      "memory": {"limit": 536870912},
      "cpu": {"quota": 150000, "period": 100000}
    },
    "namespaces": ["pid", "network", "ipc", "uts", "mount"]
  }
}
```

**Step 3:** runc creates cgroups:
```bash
mkdir /sys/fs/cgroup/cpu/docker/<container-id>
echo 150000 > .../cpu.cfs_quota_us
echo 100000 > .../cpu.cfs_period_us

mkdir /sys/fs/cgroup/memory/docker/<container-id>
echo 536870912 > .../memory.limit_in_bytes
```

**Step 4:** runc creates namespaces:
```c
clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | 
      CLONE_NEWUTS | CLONE_NEWIPC)
```

**Step 5:** Assigns process to cgroups and starts application

---

## Kubernetes and cgroups

### Architecture with cgroups

```
┌────────────────────────────────────────────────┐
│                Kubernetes Node                 │
├────────────────────────────────────────────────┤
│  kubelet (Node Agent)                          │
│  ├─ Watches API server for pod specs           │
│  ├─ Translates QoS → cgroup settings           │
│  └─ Monitors via cAdvisor                      │
├────────────────────────────────────────────────┤
│  Container Runtime (containerd/CRI-O)          │
├────────────────────────────────────────────────┤
│  OCI Runtime (runc)                            │
│  └─ Creates cgroup hierarchy per container     │
├────────────────────────────────────────────────┤
│  Linux Kernel (cgroups)                        │
│  /sys/fs/cgroup/kubepods/                      │
│    ├── burstable/                              │
│    ├── besteffort/                             │
│    └── pod<uid>/                               │
└────────────────────────────────────────────────┘
```

---

### Kubernetes QoS Classes

#### 1. Guaranteed (Highest Priority)

```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "1000m"
  limits:
    memory: "1Gi"    # limits == requests
    cpu: "1000m"
```

**Resulting cgroups:**
```
cpu.max: "100000 100000"    # 1 CPU
memory.max: 1073741824       # 1Gi
oom_score_adj: -997          # Least likely OOM kill
```

---

#### 2. Burstable (Medium Priority)

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "2Gi"    # limits > requests
    cpu: "2000m"
```

**Resulting cgroups:**
```
cpu.max: "200000 100000"    # Can burst to 2 CPUs
cpu.weight: 512              # Based on request
memory.max: 2147483648       # 2Gi hard limit
memory.high: 536870912       # 512Mi soft limit
oom_score_adj: 999           # Medium OOM priority
```

---

#### 3. BestEffort (Lowest Priority)

```yaml
# No resources specified
```

**Resulting cgroups:**
```
cpu.max: "max 100000"     # No CPU limit
memory.max: max           # No memory limit
cpu.weight: 2             # Lowest priority
oom_score_adj: 1000       # First to be OOM killed
```

---

### Kubernetes cgroup Hierarchy

```
/sys/fs/cgroup/
└── kubepods/
    ├── burstable/
    │   └── pod<uid>/
    │       └── <container-id>/
    │           ├── cpu.max
    │           ├── cpu.weight
    │           ├── memory.max
    │           └── memory.high
    ├── besteffort/
    │   └── pod<uid>/
    │       └── <container-id>/
    └── pod<uid>/             # Guaranteed pods
        └── <container-id>/
```

---

## Practical Examples

### Inspecting Container cgroups

```bash
# Run a container
docker run -d --name test \
  --cpus=0.5 \
  --memory=256m \
  --pids-limit=50 \
  nginx

# Get container ID
CONTAINER_ID=$(docker inspect -f '{{.Id}}' test)

# Examine CPU cgroup
cat /sys/fs/cgroup/cpu/docker/$CONTAINER_ID/cpu.cfs_quota_us
# Output: 50000 (0.5 CPU)

# Examine memory cgroup
cat /sys/fs/cgroup/memory/docker/$CONTAINER_ID/memory.limit_in_bytes
# Output: 268435456 (256MB)

cat /sys/fs/cgroup/memory/docker/$CONTAINER_ID/memory.usage_in_bytes
# Output: Current usage

# Examine PIDs cgroup
cat /sys/fs/cgroup/pids/docker/$CONTAINER_ID/pids.max
# Output: 50
```

### Creating Manual cgroups

```bash
# Create cgroup
mkdir /sys/fs/cgroup/cpu/myapp
mkdir /sys/fs/cgroup/memory/myapp

# Set 25% CPU limit
echo 25000 > /sys/fs/cgroup/cpu/myapp/cpu.cfs_quota_us
echo 100000 > /sys/fs/cgroup/cpu/myapp/cpu.cfs_period_us

# Set 1GB memory limit
echo 1073741824 > /sys/fs/cgroup/memory/myapp/memory.limit_in_bytes

# Assign current shell
echo $$ > /sys/fs/cgroup/cpu/myapp/cgroup.procs
echo $$ > /sys/fs/cgroup/memory/myapp/cgroup.procs

# Now run application (inherits limits)
./my_application
```

### Monitoring Commands

```bash
# View cgroup hierarchy
systemd-cgls

# Real-time monitoring
systemd-cgtop

# Docker stats (reads cgroups)
docker stats

# Kubernetes (via cAdvisor)
kubectl top pods
```

---

## Complete Flow: Pod to cgroups

```
Developer writes Pod YAML with resource requests/limits
              ↓
kubectl apply → API Server
              ↓
Scheduler assigns Pod to Node
              ↓
kubelet receives Pod spec
              ↓
kubelet calculates QoS class
              ↓
kubelet calls CRI (containerd)
              ↓
containerd calls runc with OCI spec
              ↓
runc creates cgroup hierarchy
              ↓
runc sets cpu.max, memory.max, etc.
              ↓
runc assigns container process to cgroups
              ↓
Linux kernel enforces limits
              ↓
cAdvisor monitors cgroup metrics
              ↓
Metrics exposed via kubelet → metrics-server → kubectl top
```

---

*Previous: [04_Linux_Kernel_Virtualization_Patches.md](04_Linux_Kernel_Virtualization_Patches.md)*
*Next: [06_KVM_in_Cloud_Providers.md](06_KVM_in_Cloud_Providers.md)*
