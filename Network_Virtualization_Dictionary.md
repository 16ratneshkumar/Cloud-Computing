# Network Virtualization Dictionary
> A comprehensive A–Z reference for network virtualization — covering kernel bypass, virtual interfaces, container networking, eBPF/XDP, SR-IOV, DPDK, SDN, NFV, and observability.

---

## Table of Contents

- [A — AF_XDP · ARP · API Gateway](#a)
- [B — BPF Loader · BPF Maps · BPF Verifier · BGP](#b)
- [C — Calico · Cgroups · Cilium · ClusterIP · CNI · CNI Bridge · Connect4 · CPU Pinning](#c)
- [D — DPDK · DPDK Mempool · DPDK PMD Controlled · DPDK vSwitch · DPI](#d)
- [E — eBPF · eBPF Verifier · ECMP](#e)
- [F — Flannel · Flow Table · FRR](#f)
- [G — Geneve · GRE](#g)
- [H — Hugepages · Hubble](#h)
- [I — Interface · IPVLAN · IPVS · iptables](#i)
- [K — Kernel Bypass · kube-proxy · KVM](#k)
- [L — Linux Bridge · Linux Network Stack · LLDP](#l)
- [M — MACVLAN · Memif · MTU](#m)
- [N — NAT · Network Namespace · NFV · NodePort · NUMA](#n)
- [O — OVN · OVS · OVS-DPDK Daemon · OpenFlow](#o)
- [P — PMD · Pod CIDR · PF (Physical Function)](#p)
- [Q — QoS · Queue Disciplines (qdisc)](#q)
- [R — RSS · RDMA · RX/TX Ring Buffer](#r)
- [S — SDN · SPAN · SR-IOV · SR-IOV VF](#s)
- [T — TAP · TC (Traffic Control) · TUN · TC Hook](#t)
- [U — UIO/VFIO Driver](#u)
- [V — Veth Pair · Virtio · Virtio-net · Virtio-user PMD · VLAN · VPP · Vhost-net · Vhost-user · VXLAN](#v)
- [W — WireGuard](#w)
- [X — XDP Hook Point · XDP Actions](#x)
- [Quick Reference Tables](#quick-reference-tables)
- [Architecture Compositions](#architecture-compositions)

---

## A

### AF_XDP
A high-performance Linux socket family that allows userspace applications to receive and send packets at near line-rate by connecting XDP programs directly to userspace memory. Unlike standard sockets, AF_XDP bypasses the full kernel network stack. Packets land in a userspace ring buffer (UMEM) shared between the kernel XDP program and the application. Used in Suricata IDS, VPP, and custom packet generators as a middle ground between full DPDK kernel bypass and standard socket APIs — retaining kernel driver compatibility without the full stack overhead.

```
NIC → XDP program → AF_XDP socket → userspace UMEM ring
                  ↘ XDP_PASS → normal kernel stack
```

### ARP (Address Resolution Protocol)
Maps IP addresses to MAC addresses within a Layer 2 broadcast domain. In virtual networks, each network namespace, veth pair, and bridge port maintains its own ARP table. ARP flooding in large flat networks is a major scalability concern — VXLAN with BUM (Broadcast, Unknown unicast, Multicast) traffic suppression and BGP EVPN solve this by distributing MAC/IP bindings via control plane instead of data plane flooding.

### API Gateway (in NFV context)
A service function that acts as an entry point for north-south traffic in a containerized or NFV environment. From a network virtualization standpoint, it is a VNF (Virtual Network Function) that sits in the service chain, performing TLS termination, rate limiting, and routing. Traffic reaches it via a Load Balancer VIP, passes through NAT, and is forwarded into the cluster network namespace.

---

## B

### BGP (Border Gateway Protocol)
A path-vector routing protocol used to exchange network reachability information between autonomous systems. In Kubernetes networking, Calico uses BGP to advertise pod CIDRs to physical routers, enabling pod-to-pod communication across nodes without overlay encapsulation. BGP EVPN (Ethernet VPN) is widely used in modern data centers to distribute MAC and IP information for VXLAN overlays, replacing flood-and-learn with a control plane.

### BPF Loader
A userspace tool or library responsible for compiling, loading, and attaching eBPF bytecode into the kernel. It handles map creation, pinning programs to the BPF filesystem (`/sys/fs/bpf/`), and linking programs to hook points. Common implementations include `libbpf` (C library), `bpftool` (CLI), and Cilium's own loader which dynamically manages per-pod eBPF programs. The loader must negotiate with the kernel verifier — if verification fails, the program is rejected.

```
eBPF Program Lifecycle:
  Write (C) → Compile (clang/LLVM) → Load (BPF loader) → Verify (kernel verifier) → JIT → Attach to hook
```

### BPF Maps
Kernel data structures shared between eBPF programs and userspace applications. They are the persistence and communication layer for eBPF — storing connection tracking state, policy rules, service-to-backend mappings, counters, and metrics. Key map types:

| Map Type | Use Case |
|----------|----------|
| `BPF_MAP_TYPE_HASH` | Connection tracking, policy lookup |
| `BPF_MAP_TYPE_ARRAY` | Global config, per-CPU counters |
| `BPF_MAP_TYPE_LRU_HASH` | NAT state with automatic eviction |
| `BPF_MAP_TYPE_PERCPU_HASH` | High-throughput per-CPU stats |
| `BPF_MAP_TYPE_PROG_ARRAY` | Tail call dispatch tables |
| `BPF_MAP_TYPE_SOCKHASH` | Socket-level redirection (Cilium) |

### BPF Verifier
The kernel subsystem that statically analyzes every eBPF program before it is allowed to run. It performs: control-flow analysis (no unbounded loops), bounds checking (no out-of-bounds memory access), type safety (registers have tracked types), and pointer safety (no raw kernel pointer leaks to userspace). If verification fails, the program load is rejected and an error message is returned to the loader. The verifier is the security boundary that makes eBPF safe to run in kernel context.

---

## C

### Calico eBPF
Calico's high-performance native dataplane mode that replaces iptables with eBPF programs. In eBPF mode, Calico attaches programs to TC (Traffic Control) hook points on each pod interface, replacing kube-proxy for service load balancing and replacing iptables chains for network policy. It supports direct server return (DSR) for services, reducing the number of network hops for external traffic. Calico also supports a VPP dataplane for telco-grade performance, and standard Linux iptables for simpler deployments.

### Cgroups (Control Groups)
A Linux kernel mechanism that organizes processes into hierarchical groups for resource management. In container runtimes, each pod gets its own cgroup subtree controlling CPU, memory, and network bandwidth. In eBPF networking, cgroups serve as attach points — eBPF programs attached to a cgroup's network hooks (`cgroup/connect4`, `cgroup/sendmsg`, `cgroup/recvmsg`) intercept socket-level syscalls for all processes in that cgroup. Cilium uses this to implement per-pod service load balancing and socket-level policy without any packet-level overhead.

### Cilium
A Kubernetes CNI plugin built entirely on eBPF that replaces iptables-based kube-proxy with eBPF programs throughout the datapath. Architecture layers:

| Layer | Mechanism | Purpose |
|-------|-----------|---------|
| Socket | Connect4 / cgroup hooks | Service load balancing, transparent redirect |
| TC ingress/egress | eBPF at TC hook | Network policy, encryption, observability |
| XDP | eBPF at NIC driver | DDoS protection, NodePort acceleration |
| Control plane | Hubble / Cilium agent | Policy distribution, observability |

Cilium stores all service endpoints, identity-based policies, and NAT state in BPF maps, enabling O(1) policy lookups instead of O(n) iptables chain traversal.

### ClusterIP
A Kubernetes Service type that assigns a virtual IP internal to the cluster. Traffic to this IP is intercepted and load-balanced to healthy pod backends. Traditionally implemented by kube-proxy using iptables DNAT rules. Cilium replaces this with eBPF socket-level redirection via Connect4, intercepting the `connect()` syscall before the packet is even created — eliminating NAT entirely in the common case.

### CNI (Container Network Interface)
A specification and plugin ecosystem for configuring network interfaces in Linux containers. When a pod is created, the container runtime (containerd, CRI-O) calls the CNI plugin binary with a JSON config. The plugin must: create a network interface inside the pod's network namespace, assign an IP address, configure routes, and return result info. At pod deletion, it tears everything down. CNI defines the contract — plugins implement it with any technology (veth+bridge, eBPF, SR-IOV, IPVLAN, etc.).

```
kubelet → containerd → CNI plugin binary → creates veth + assigns IP + configures routes
```

### CNI Bridge
The reference CNI plugin that creates a Linux bridge on the host (`cni0` by default) and connects each pod via a veth pair. One end of the veth goes into the pod namespace as `eth0`, the other attaches to the bridge on the host. Simple and dependency-free but lacks network policy, high performance datapaths, or overlay networking capabilities. Typically replaced by Calico, Cilium, or Flannel in real clusters.

```
Pod NS: eth0 (veth0) <──veth──> host: vethXXXX ──> cni0 (Linux Bridge) ──> eth0 (host NIC)
```

### Connect4
An eBPF program type that hooks into the `connect()` system call for IPv4 TCP/UDP sockets at the cgroup level. Used by Cilium for transparent, zero-NAT service load balancing. When a pod issues `connect()` to a ClusterIP, the eBPF program intercepts it and rewrites the destination to a selected backend pod IP — before any packet is generated. The application is completely unaware. This is significantly more efficient than iptables DNAT because it operates at the socket layer, with no per-packet overhead.

### CPU Pinning
The practice of dedicating specific CPU cores exclusively to packet processing threads, preventing the OS scheduler from moving them. Essential for DPDK PMD threads — unpredictable scheduling would cause gaps in polling and packet drops. Configured via `isolcpus` kernel parameter and DPDK's EAL (Environment Abstraction Layer). Also used with OVS-DPDK to pin PMD threads to specific cores on the correct NUMA node.

---

## D

### DPDK (Data Plane Development Kit)
A collection of userspace libraries, drivers, and tools enabling applications to take direct control of NIC hardware and process packets entirely in userspace, bypassing the Linux kernel network stack. The core performance gains come from: Poll Mode Drivers (no interrupt overhead), hugepage-backed mempools (no TLB misses), zero-copy packet processing, and per-core lock-free queues.

```
Normal kernel path:  NIC → kernel driver (IRQ) → sk_buff → netfilter → socket → app
DPDK path:           NIC → VFIO/UIO → DPDK PMD (polling) → mbuf → userspace app
                     (no kernel involvement after initial setup)
```

Key use cases: telco 5G UPF (User Plane Function), vRouter, firewall VNFs, OVS-DPDK, and high-frequency trading packet capture.

### DPDK EAL (Environment Abstraction Layer)
The foundation layer of DPDK that abstracts hardware details and OS specifics. EAL handles: hugepage allocation, CPU affinity and NUMA topology, PCI device probing and binding, inter-process communication, and logging. Every DPDK application initializes EAL first via `rte_eal_init()` before accessing any hardware.

### DPDK Mempool
A pre-allocated pool of fixed-size memory buffers (`rte_mbuf` structures) backed by hugepages. Used to store packet data for NIC DMA operations. The pool is created at startup, avoiding runtime memory allocation in the hot path. When the PMD receives packets, it fills mbufs from the pool; when processing is done, mbufs are returned to the pool. Per-CPU local caches reduce contention in multi-core scenarios.

```
Hugepage memory region
  └── Mempool
        ├── mbuf[0]: packet data + metadata
        ├── mbuf[1]: packet data + metadata
        └── mbuf[N]: packet data + metadata
```

### DPDK PMD Controlled
Describes a network interface bound to a DPDK Poll Mode Driver rather than the Linux kernel. The interface is invisible to the kernel network stack (`ip link` won't show it). Only the DPDK application can send and receive on it. Applies to both physical NICs (bound via `vfio-pci`) and virtual interfaces (`virtio-user`, `memif`, `vhost-user`). The kernel driver must be unloaded and replaced with `vfio-pci` or `uio_pci_generic` before DPDK can take control.

### DPDK vSwitch
A software switch whose entire dataplane runs within DPDK userspace. Packets flow: Physical NIC (via PMD) → vSwitch logic (flow lookup, forwarding) → destination PMD (VM virtqueue, another NIC, or memif). No kernel datapath is involved. OVS-DPDK is the most widely deployed DPDK vSwitch; VPP is another. Achieves millions of packets per second per core on commodity hardware.

### DPI (Deep Packet Inspection)
Analysis of packet payload beyond L2–L4 headers — examining application-layer content to identify protocols (HTTP, DNS, TLS fingerprinting), detect intrusions (IDS/IPS), enforce content policies, or perform lawful intercept. In virtual network architectures, DPI engines are inserted as service functions in an NFV service chain. Traffic is steered to them via SPAN (passive copy) or as an inline bump-in-the-wire function. Performance-sensitive DPI implementations use DPDK or AF_XDP to handle 10/40/100 Gbps line rates.

---

## E

### eBPF (Extended Berkeley Packet Filter)
A Linux kernel subsystem enabling safe, sandboxed programs to run inside the kernel without modifying kernel source or loading modules. Programs are written in a restricted C subset, compiled to eBPF bytecode by clang/LLVM, verified for safety by the kernel verifier, then JIT-compiled to native machine code. In networking, eBPF hooks into the packet processing pipeline at multiple points (XDP, TC, sockets, cgroups) enabling packet filtering, NAT, load balancing, observability, and security policy with minimal overhead. The foundation of Cilium, Calico eBPF mode, and modern Linux firewalling.

### eBPF Verifier
See [BPF Verifier](#bpf-verifier). The kernel-side gatekeeper that ensures no eBPF program can crash the kernel, access arbitrary memory, or execute indefinitely. As programs grow more complex, the verifier's complexity analysis becomes a practical constraint — programs with many conditional branches or complex pointer arithmetic may hit verifier limits even if logically correct.

### ECMP (Equal-Cost Multi-Path)
A routing technique that distributes traffic across multiple equal-cost paths to a destination, providing both load balancing and redundancy. Used in Kubernetes with Calico BGP: multiple nodes advertise the same pod CIDR prefix, and upstream routers distribute flows across all of them using ECMP hashing. Combined with XDP or IPVS, ECMP can enable highly scalable stateless load balancing at the network layer.

---

## F

### Flannel
A simple, lightweight CNI plugin focused purely on providing pod-to-pod connectivity by giving each node a subnet and encapsulating cross-node traffic. VXLAN is the default backend; UDP and host-gw (pure routing, no overlay) are alternatives. Flannel does not implement network policy — commonly paired with Calico (for policy only, "Flannel + Calico" = Canal) or replaced entirely by Cilium or Calico in production. Easy to operate and widely supported.

### Flow Table
A data structure in a virtual switch (OVS, DPDK vSwitch) that stores forwarding rules. When a packet arrives, its header fields are matched against the flow table; a match yields an action (forward, drop, modify, encapsulate). In OVS, the kernel datapath caches flow table entries from the userspace `ovs-vswitchd` daemon — the first packet of a flow goes to userspace (slow path), subsequent packets are handled by the cached kernel entry (fast path). DPDK vSwitches keep the entire flow table in userspace.

### FRR (Free Range Routing)
An open-source routing suite (BGP, OSPF, IS-IS, PIM, etc.) commonly used in Kubernetes and NFV environments to handle dynamic routing. Calico uses BIRD (a similar routing daemon) or optionally integrates with FRR for BGP route distribution. In NFV architectures, FRR runs as a VNF inside a container, participating in the physical network's routing protocol to advertise VNF prefixes.

---

## G

### Geneve (Generic Network Virtualization Encapsulation)
An overlay tunnel encapsulation that improves upon VXLAN by supporting variable-length metadata headers. Network control planes can embed arbitrary metadata (policy IDs, security labels, etc.) into Geneve headers, enabling richer network virtualization features without additional processing layers. OVN (Open Virtual Network) uses Geneve as its default tunnel encapsulation for overlay networking.

### GRE (Generic Routing Encapsulation)
A tunneling protocol that encapsulates any network layer protocol inside IP packets. Simpler than VXLAN (no UDP header, no port number), making it easier to route but harder to load-balance across ECMP paths (ECMP uses L4 port numbers for hashing). OVS supports GRE tunnels. Flannel supports GRE as a backend option.

---

## H

### Hugepages
Memory pages larger than the OS default of 4 KB — typically 2 MB (standard hugepages) or 1 GB (1G hugepages). DPDK requires hugepages because packet buffer regions must be large, contiguous, physically pinned, and non-swappable for reliable NIC DMA. Without hugepages, the TLB would be overwhelmed with entries for large mempool regions, causing frequent TLB misses and performance degradation. Configured in the OS before DPDK applications start.

```bash
# Configure 1024 × 2 MB hugepages
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# Mount hugetlbfs
mount -t hugetlbfs nodev /dev/hugepages
```

### Hubble
The observability layer of Cilium. A distributed networking and security observability platform that provides real-time visibility into network flows, DNS queries, HTTP transactions, and dropped packets across a Kubernetes cluster. Hubble reads data directly from Cilium's eBPF maps and perf ring buffers — zero additional overhead on the data path. Exposes data via a CLI (`hubble observe`), UI, and Prometheus metrics.

---

## I

### Interface (Network Interface)
A logical or physical attachment point where a system connects to a network. In virtualization contexts, "interface" is an abstraction that may be realized as: a physical NIC port, a virtual NIC (vNIC) presented to a VM by the hypervisor, a veth pair endpoint, a TAP device file descriptor, a DPDK PMD port, an SR-IOV Virtual Function, or a memif shared-memory endpoint — each with different performance, isolation, and management characteristics.

### IPVLAN
A Linux virtual interface type that shares the parent interface's MAC address but gives each sub-interface its own IP address. Unlike MACVLAN (which assigns unique MACs), IPVLAN reduces ARP overhead and works in environments where the switch limits the number of MACs per port (common in cloud environments). Two modes: L2 (stays in same broadcast domain) and L3 (each sub-interface is its own L3 network, no ARP between them). Used as a CNI mechanism for high-density pod networking where MAC address limits are a concern.

### IPVS (IP Virtual Server)
A Linux kernel load-balancing module that operates at L4 using netfilter hooks. Significantly more scalable than iptables for Kubernetes Service load balancing — IPVS uses hash tables (O(1) lookup) vs. iptables linear rule chains (O(n) traversal). kube-proxy can run in IPVS mode as an intermediate step between iptables and full eBPF. Supports multiple scheduling algorithms: round-robin, least-connection, source-hash, etc.

### iptables
A Linux firewall and packet mangling framework built on the Netfilter hook system. Traditionally the backbone of Kubernetes Service networking (kube-proxy writes DNAT rules for each Service endpoint). Scales poorly: adding a new endpoint requires rewriting the full ruleset, and rule traversal is O(n) per packet. Being replaced in modern deployments by eBPF (Cilium, Calico eBPF) or IPVS. Still widely used for basic NAT and firewalling in traditional virtual networking.

---

## K

### Kernel Bypass
A design philosophy where packet processing is moved entirely out of the Linux kernel network stack into userspace. Eliminates: interrupt handling overhead, kernel-to-userspace context switches, sk_buff memory allocation per-packet, and netfilter/iptables traversal. Achieved via DPDK (full bypass via UIO/VFIO), AF_XDP (partial bypass via shared memory ring), or SR-IOV VF passthrough (hardware-level bypass). The trade-off is loss of kernel networking features (iptables, routing, tc) and higher operational complexity.

### kube-proxy
The Kubernetes component (runs on every node) responsible for implementing Service networking. Watches the Kubernetes API for Service and Endpoint changes and programs the local node's network accordingly. Three modes:

| Mode | Mechanism | Scale | Notes |
|------|-----------|-------|-------|
| iptables | netfilter DNAT rules | Hundreds of services | Default, widely used |
| IPVS | L4 kernel load balancer | Thousands of services | Better performance |
| eBPF (Cilium) | BPF maps + socket hooks | Tens of thousands | kube-proxy replaced entirely |

### KVM (Kernel-based Virtual Machine)
A Linux kernel module that turns the kernel into a Type-1 hypervisor. KVM allows running hardware-accelerated VMs. From a network virtualization standpoint, each KVM VM is connected via a TAP device to the host network — the hypervisor (QEMU) reads/writes Ethernet frames via the TAP file descriptor and presents them to the VM as a virtio-net or emulated NIC. High-performance VM networking replaces this with vhost-net, vhost-user, or SR-IOV VF passthrough.

---

## L

### Linux Bridge
A kernel-level L2 software switch (`brctl` / `ip link add type bridge`). Learns source MAC addresses, builds a forwarding table, and forwards frames between attached interfaces. Docker's default `docker0` bridge, Kubernetes CNI Bridge plugin, and KVM/libvirt default networking all use Linux bridges. Slower than OVS-DPDK or eBPF but zero additional dependencies — the baseline for understanding all more sophisticated virtual switching.

### Linux Network Stack
The complete packet processing pipeline built into the Linux kernel. Inbound path: NIC driver → NAPI polling → sk_buff allocation → netfilter (iptables) PRE_ROUTING → IP routing decision → netfilter LOCAL_IN or FORWARD → transport layer (TCP/UDP) → socket receive buffer. Outbound path reverses this. Provides rich, well-understood features but adds latency (typical: 50–200 µs end-to-end in a VM) compared to kernel bypass approaches (1–5 µs with DPDK).

### LLDP (Link Layer Discovery Protocol)
An L2 protocol that allows network devices to advertise their identity, capabilities, and neighbors. In virtual environments, LLDP passthrough allows the hypervisor or virtual switch to forward LLDP frames from physical switches into VMs, giving network management systems visibility into the physical-to-virtual topology. OVS supports LLDP passthrough configurations.

---

## M

### MACVLAN
A Linux virtual interface type that creates multiple virtual interfaces on top of a single physical interface, each with its own unique MAC address. The kernel performs L2 demultiplexing based on destination MAC. Used in container networking to give containers MAC addresses on the same physical network as the host, without bridges. Four modes: bridge (local communication between macvlans), VEPA (traffic hairpins through physical switch), passthru (single VM gets full interface), private (no local communication).

### Memif (Memory Interface)
A shared-memory packet I/O mechanism used between userspace processes. Two applications share a ring buffer memory region; packets are passed by reference (descriptor exchange) with no data copying and no kernel involvement. Control information is exchanged over a Unix domain socket. Primary use: connecting VPP instances, or connecting a VPP/DPDK application to another NFV process (e.g., Snort, DPDK app) at line rate. Dramatically faster than sending packets through a TAP device or kernel socket.

```
[ NFV App A ] ←── shared memory ring (zero-copy) ──→ [ NFV App B ]
                           memif
```

### MTU (Maximum Transmission Unit)
The largest packet size (in bytes) that can be transmitted on a network link without fragmentation. In virtual networks, overlay encapsulations add bytes to every packet — VXLAN adds 50 bytes (UDP + VXLAN headers). If the physical network MTU is 1500, overlay traffic must use 1450-byte inner MTU to avoid fragmentation. Misconfigured MTUs cause subtle performance issues (fragmentation, TCP retransmits, slow large transfers) and are a common source of bugs in overlay networks. Jumbo frames (MTU 9000) are commonly configured on physical networks to give overlay protocols headroom.

---

## N

### NAT (Network Address Translation)
Rewriting source or destination IP addresses (and ports) in packet headers as they transit a gateway. In container networking: pods have private IPs (e.g., `10.0.0.0/8`); NAT translates these to the node's public IP for external access (SNAT). For inbound service traffic, NAT translates the Service ClusterIP to a pod IP (DNAT). Implemented by iptables (conntrack), IPVS, or eBPF. Cilium's Connect4 hook eliminates NAT for intra-cluster service traffic entirely by doing socket-level redirection.

### Network Namespace
A Linux kernel primitive providing full isolation of the network stack. Each namespace has its own: network interfaces, IP addresses, routing tables, iptables rules, conntrack table, and sockets. The root namespace is the host's network; containers and pods each get their own namespace. Veth pairs are the standard cable connecting namespaces. Processes can only see interfaces in their own namespace (unless they have `CAP_NET_ADMIN` and access the root namespace). The `ip netns` command manages namespaces; `/var/run/netns/` stores their file handles.

### NFV (Network Function Virtualization)
An architecture that implements network functions (firewalls, load balancers, WAN optimizers, DPI engines, IDS/IPS, 5G UPF) as software running on commodity x86 servers rather than dedicated proprietary appliances. VNFs (Virtual Network Functions) are chained together as a Service Function Chain (SFC). High-performance NFV stacks use DPDK for packet I/O, SR-IOV for hardware isolation, memif for inter-VNF communication, and OVS-DPDK or VPP as the vSwitch connecting them.

### NodePort
A Kubernetes Service type that exposes a service on a static port across all cluster nodes. External clients connect to `<NodeIP>:<NodePort>`; the node NATs the traffic to a backend pod. Implemented by kube-proxy (iptables/IPVS) or Cilium (XDP + eBPF for the external path). Cilium can process NodePort packets at the XDP layer — before they even enter the kernel stack — achieving extremely low-latency service entry.

### NUMA (Non-Uniform Memory Access)
A multi-socket CPU memory architecture where each CPU socket has local memory banks (fast, ~100ns) and can access remote socket memory (slow, ~200–300ns). DPDK is rigorously NUMA-aware: mempools, huge pages, and PMD threads must all be allocated on the same NUMA node as the NIC to avoid cross-socket memory latency. OVS-DPDK automatically detects NUMA topology and allocates resources accordingly. Ignoring NUMA in a DPDK deployment causes silent, hard-to-diagnose performance degradation.

---

## O

### OVN (Open Virtual Network)
A higher-level network virtualization control plane built on top of OVS. OVN provides logical networking abstractions: logical switches, logical routers, ACLs, NAT, and load balancing — independent of the physical topology. OVN translates logical network intent into OVS flow rules. Used as the default network provider for OpenStack (since Queens release) and as an option for Kubernetes (via ovn-kubernetes CNI). Uses Geneve encapsulation for tunnel overlay.

### OVS (Open vSwitch)
A production-grade, programmable multi-layer virtual switch designed for virtualized environments. Key capabilities: OpenFlow for SDN control plane integration, VLAN tagging, VXLAN/GRE/Geneve tunnel endpoints, quality of service (QoS), traffic metering, and bond interfaces. Used in OpenStack, Kubernetes (with OVN), and telco NFV. OVS architecture:

| Component | Role |
|-----------|------|
| `ovs-vswitchd` | Main daemon, slow-path, OpenFlow processing |
| `ovsdb-server` | Configuration and state database |
| Kernel datapath | Fast-path flow cache in kernel |
| DPDK datapath | Userspace fast-path (OVS-DPDK mode) |

### OVS-DPDK Daemon
OVS running with its datapath powered by DPDK. `ovs-vswitchd` uses DPDK PMD threads to handle all packet I/O in userspace. The kernel datapath is unused. PMD threads poll vhost-user ports (for VMs), DPDK physical ports, and internal ports. The result is dramatically higher throughput (10–100 Gbps+) and lower latency than kernel OVS. Requires: DPDK-compatible NIC, hugepages, CPU pinning, and NUMA-aware memory configuration.

### OpenFlow
A protocol that enables an external SDN controller to program flow tables in a virtual or physical switch. OVS is the most prominent OpenFlow-capable virtual switch. The controller (e.g., OpenDaylight, ONOS, Ryu) pushes flow rules to OVS; OVS executes them on every matching packet. Separates the control plane (who decides what happens to packets) from the data plane (who executes those decisions), enabling centralized, programmable network management.

---

## P

### PMD (Poll Mode Driver)
DPDK's fundamental driver model. Rather than using hardware interrupts (which cause context switches and scheduling latency), a PMD dedicates one or more CPU cores to continuously poll the NIC's RX descriptor rings for new packets. When a packet arrives in the ring, the PMD processes it immediately. Eliminates interrupt latency at the cost of CPU — PMD cores run at 100% utilization even when the network is idle. Multiple PMD threads can be pinned to different cores, each handling separate RX/TX queues.

### PF (Physical Function)
In SR-IOV terminology, the Physical Function is the full-featured PCIe device (the actual NIC). It has complete access to hardware configuration, can create and manage Virtual Functions, and is managed by the host OS via the standard NIC driver (e.g., `i40e`, `mlx5_core`). The PF driver exposes a sysfs interface to create VFs: `echo 4 > /sys/bus/pci/devices/<PCI_ADDR>/sriov_numvfs`.

### Pod CIDR
The IP address range assigned to a Kubernetes node from which pod IPs are allocated. Each node gets a unique subnet (e.g., `10.244.1.0/24`). The CNI plugin assigns individual pod IPs from this range. Pod CIDRs from all nodes form the cluster's pod network (e.g., `10.244.0.0/16`). In Calico BGP mode, each node advertises its pod CIDR to physical routers via BGP, enabling routable pod IPs without overlay encapsulation.

---

## Q

### QoS (Quality of Service)
Mechanisms to manage bandwidth, latency, and packet loss characteristics for different traffic flows. In Linux, QoS is implemented via `tc` (Traffic Control) with queueing disciplines (qdiscs). In OVS, QoS is configured per-port and per-queue using meters and DSCP marking. In DPDK, QoS is implemented via the `rte_sched` library with hierarchical scheduling. Critical in NFV environments to ensure SLA compliance for latency-sensitive workloads (VoIP, 5G control plane) sharing infrastructure with bulk traffic.

### Queue Disciplines (qdisc)
Linux kernel's packet scheduling framework attached to network interfaces, determining how packets are queued and dequeued. Key qdiscs:

| qdisc | Type | Use Case |
|-------|------|----------|
| `pfifo_fast` | Classless | Default, 3-band priority |
| `htb` (Hierarchy Token Bucket) | Classful | Rate limiting with burst |
| `fq_codel` | Classless | Low-latency, AQM |
| `tbf` (Token Bucket Filter) | Classless | Traffic shaping |
| `mqprio` | Classful | Multi-queue NIC mapping |

In Kubernetes, CNI plugins may attach qdiscs to veth interfaces to enforce per-pod bandwidth limits.

---

## R

### RDMA (Remote Direct Memory Access)
A technology allowing one computer to directly read/write another's memory without involving either system's OS or CPU. Used in high-performance computing and increasingly in storage (NVMe-oF) and AI/ML training clusters. From a networking standpoint, RDMA over Converged Ethernet (RoCE v2) requires lossless Ethernet (PFC — Priority Flow Control) and uses SR-IOV VFs for hardware isolation. DPDK supports RDMA through specific PMDs.

### RSS (Receive Side Scaling)
A NIC hardware feature that distributes incoming packets across multiple RX queues using a hash of the flow 5-tuple (src IP, dst IP, src port, dst port, protocol). Allows multi-queue NICs to parallelize packet reception across multiple CPU cores. DPDK PMDs configure RSS at initialization. OVS-DPDK relies on RSS to spread load across multiple PMD threads. Without RSS, all packets arrive in one queue, creating a single-core bottleneck regardless of how many PMD threads are running.

### RX/TX Ring Buffer
Fixed-size circular queues of DMA descriptors shared between the NIC hardware and the driver (or DPDK PMD). The NIC writes received packet data into memory locations pointed to by RX descriptors; the driver reads these descriptors to find received packets. For TX, the driver places descriptors pointing to packet data; the NIC reads them and transmits. Ring buffer size (number of descriptors) is configurable — larger rings reduce packet drop under burst traffic at the cost of memory and latency.

---

## S

### SDN (Software Defined Networking)
An architecture that decouples the network control plane (decisions about where traffic goes) from the data plane (the actual packet forwarding). The control plane runs as software on servers (SDN controller); the data plane runs on switches/routers programmed by the controller via protocols like OpenFlow. In virtual networking, OVS is the canonical SDN-capable vSwitch. Cilium and OVN are examples of modern SDN control planes for container environments.

### SPAN (Switched Port Analyzer)
Port mirroring that copies traffic from one or more source ports/interfaces to a designated monitor port, without affecting the original traffic flow. In virtual networks, OVS supports SPAN via `ovs-vsctl` mirror configuration. Used to feed traffic to: DPI engines, IDS/IPS sensors, packet capture tools (Wireshark), and network monitoring probes. Does not slow down the mirrored traffic path since it operates on a copy.

### SR-IOV (Single Root I/O Virtualization)
A PCIe standard that allows a single physical NIC (Physical Function) to expose multiple Virtual Functions — lightweight PCIe devices with their own hardware queues, interrupts, and DMA capabilities. Each VF can be assigned directly to a VM or container via VFIO passthrough. Packets flow directly between the VF hardware and the guest, bypassing any software vSwitch. Provides near line-rate performance with hardware-enforced isolation.

```
Physical NIC (PF)  ← managed by host driver
  ├── VF 0 ──VFIO passthrough──→ VM / Pod A (dedicated hardware queue)
  ├── VF 1 ──VFIO passthrough──→ VM / Pod B (dedicated hardware queue)
  ├── VF 2 ──VFIO passthrough──→ VM / Pod C (dedicated hardware queue)
  └── VF N ──...
```

Trade-offs: No live VM migration, VF traffic not visible to host vSwitch (monitoring harder), VF count limited by NIC hardware.

### SR-IOV VF (Virtual Function)
One lightweight virtual PCIe device created by an SR-IOV Physical Function. A VF has its own TX/RX queues, memory-mapped I/O, and interrupt lines, but shares the physical link and silicon with the PF and other VFs. VF isolation is enforced by the NIC hardware's internal switch. VFs assigned to VMs via VFIO passthrough give near line-rate networking because packets go: physical wire → NIC internal switch → VF hardware queue → VM memory (DMA) — with no software forwarding hop.

---

## T

### TAP (Terminal Access Point)
A kernel virtual network interface that operates at Layer 2 (Ethernet frames). A userspace process opens `/dev/net/tun`, creates a TAP interface, and receives raw Ethernet frames via read()/write() on the file descriptor. QEMU/KVM uses TAP to connect each VM's virtio-net NIC to the host network — the VM sends Ethernet frames → QEMU reads them from the TAP FD → forwards to Linux bridge or OVS. Can be replaced by vhost-net (kernel acceleration) or vhost-user (DPDK acceleration) for performance.

### TC (Traffic Control) Hook
A Linux kernel hook point in the network stack where eBPF programs can be attached to both ingress and egress paths of any interface. Located after XDP (after sk_buff allocation) but before routing decisions (ingress) or after routing (egress). TC eBPF programs have access to the full sk_buff and can read/modify any header field, perform map lookups, and redirect packets. Cilium attaches its network policy enforcement and encryption programs at TC hooks on pod veth interfaces.

```
Ingress: NIC → XDP hook → sk_buff alloc → TC ingress hook → IP stack → socket
Egress:  socket → IP stack → TC egress hook → NIC driver → wire
```

### TUN (Network Tunnel)
A kernel virtual network interface operating at Layer 3 (IP packets). Like TAP but delivers raw IP packets (no Ethernet headers) to the userspace process. The kernel routes IP datagrams into the TUN interface; the userspace application reads them, typically encrypts them, and sends them through a real interface as UDP or TCP encapsulation. WireGuard (in kernelspace), OpenVPN, and similar VPN tools historically used TUN interfaces.

### TC Hook
See [TC (Traffic Control) Hook](#tc-traffic-control-hook) above.

---

## U

### UIO/VFIO Driver
Two Linux kernel driver frameworks enabling DPDK to take ownership of NIC hardware:

| Driver | Full Name | IOMMU Required | Security | Use Case |
|--------|-----------|---------------|----------|----------|
| `uio_pci_generic` | Userspace I/O | No | Lower | Dev/test, older systems |
| `igb_uio` | Intel UIO | No | Lower | Intel NICs specifically |
| `vfio-pci` | Virtual Function I/O | Yes (recommended) | High | Production DPDK |

VFIO with IOMMU enabled (`intel_iommu=on` in kernel cmdline) prevents a compromised DPDK application from performing DMA to arbitrary physical memory addresses — an important security boundary in multi-tenant NFV deployments. After binding a NIC to `vfio-pci`, the kernel no longer owns the device.

---

## V

### Veth Pair
A virtual Ethernet cable consisting of two linked kernel network interfaces. Anything written to one end appears on the other. The fundamental building block of container networking: one end lives inside the container's network namespace (`eth0`); the other lives on the host and connects to a bridge, OVS, or is configured for routing. Creating a veth pair: `ip link add veth0 type veth peer name veth1`. Deletion of either end deletes both.

```
[ Container Network Namespace ]       [ Host Network Namespace ]
         eth0 (veth0)           ←──→         vethXXXX
                                            ↓
                                     Linux Bridge / OVS / routes
```

### Virtio
A paravirtualization standard for I/O devices (block, network, console, crypto, etc.) between a hypervisor and guest VMs. Instead of emulating real hardware (e.g., Intel e1000), the hypervisor exposes a virtio device and the guest runs a virtio driver — both sides are aware of virtualization. Communication uses shared-memory ring buffers (virtqueues: split ring or packed ring in modern virtio 1.1+). Defined by the OASIS Virtio spec. The foundation of all high-performance VM networking (virtio-net, vhost-net, vhost-user).

### Virtio-net
The network device implementation of the Virtio spec inside a guest VM. The guest virtio-net driver communicates with the host via two virtqueues: one TX queue (guest → host) and one RX queue (host → guest). Far faster than emulated NICs (no hardware register emulation) but the bottleneck is the data path between guest and host. Performance evolution: virtio-net (QEMU userspace) → vhost-net (kernel acceleration) → vhost-user (DPDK acceleration).

### Virtio-user PMD
A DPDK Poll Mode Driver that speaks the virtio protocol over a Unix domain socket in userspace. Allows a DPDK application to connect to a vhost-user backend (OVS-DPDK, VPP) as if it were a VM guest — but from a container or bare-metal process rather than a VM. Enables high-speed packet I/O between a DPDK application and a DPDK vSwitch without any kernel involvement. Used in container NFV scenarios to achieve VM-equivalent performance.

### VLAN (Virtual LAN)
L2 network segmentation using IEEE 802.1Q tags — a 4-byte tag (12-bit VLAN ID = 4096 possible VLANs) inserted into Ethernet frames. A single physical link can carry traffic for multiple VLANs. Switches and virtual switches demultiplex frames by VLAN ID. OVS, Linux bridge, and physical NICs all support VLAN trunking. Used in multi-tenant virtual environments to isolate tenant broadcast domains sharing the same physical infrastructure. VXLAN was created to overcome the 4096 VLAN limit (VXLAN provides 16 million VNI identifiers).

### VPP (Vector Packet Processor)
An open-source (FD.io / DPDK ecosystem), high-performance userspace packet processing framework. Processes packets in vectors (batches of 32–256 packets) rather than one at a time, amortizing per-packet overhead across the vector. VPP can run as a high-performance vSwitch, router, firewall, or NAT engine. Supports: memif, vhost-user, DPDK PMDs, VXLAN, GRE, IPsec, segment routing, and more. Telco operators use VPP for 5G UPF implementations.

### Vhost-net
A Linux kernel module that accelerates virtio-net networking by moving the host-side data plane from QEMU userspace into the kernel. With vhost-net, the guest TX/RX virtqueues are serviced directly by kernel threads rather than QEMU, eliminating one userspace↔kernel boundary crossing per packet. Significant latency reduction over pure QEMU virtio-net. Can be combined with TAP for final delivery to the host network. Superseded in high-performance scenarios by vhost-user (fully userspace, DPDK-backed).

```
Without vhost-net: VM ↔ virtqueue ↔ QEMU (user) ↔ syscall ↔ kernel ↔ TAP ↔ bridge
With vhost-net:    VM ↔ virtqueue ↔ vhost-net (kernel) ↔ TAP ↔ bridge   [fewer crossings]
```

### Vhost-user
Like vhost-net but the backend lives entirely in a DPDK userspace process. The VMM (QEMU) and the vSwitch (OVS-DPDK, VPP) establish a shared memory region via a Unix domain socket; virtqueues are mapped into this shared memory. Packets pass between the VM and the vSwitch via DMA into shared hugepage memory — no kernel involvement in the data path. Achieves near line-rate VM networking (10–100 Gbps). The standard high-performance VM networking mechanism in telco NFV.

```
VM ↔ virtqueue ↔ shared hugepage memory ↔ vhost-user backend (OVS-DPDK / VPP)
                    (no kernel crossing in data path)
```

### VXLAN (Virtual Extensible LAN)
An overlay network encapsulation protocol that tunnels L2 Ethernet frames inside UDP packets (destination port 4789). A 24-bit VNI (VXLAN Network Identifier) provides 16 million logical network segments (vs. 4096 for VLANs). The VTEP (VXLAN Tunnel End Point) — running in OVS, Linux kernel, or NIC hardware — encapsulates/decapsulates packets. Widely used in Kubernetes overlay networks (Flannel, Calico VXLAN mode), OpenStack, and data center fabrics.

```
Inner Ethernet frame:
  [ Outer ETH | Outer IP | UDP (4789) | VXLAN header (VNI) | Inner ETH | Inner IP | payload ]
```

---

## W

### WireGuard
A modern, high-performance VPN protocol and Linux kernel module. Uses state-of-the-art cryptography (ChaCha20, Poly1305, Curve25519) with a very small codebase (~4000 lines vs. OpenVPN's ~600,000). Implemented as a kernel module using TUN interfaces. Cilium uses WireGuard as its transparent encryption backend — encrypting pod-to-pod traffic across nodes without any application awareness, with minimal performance overhead compared to IPsec.

---

## X

### XDP (eXpress Data Path)
An eBPF hook point at the earliest possible moment in the Linux NIC driver — before the kernel allocates an sk_buff, before NAPI processing, before any packet buffers are set up. Enables very high-speed packet processing at near-DPDK performance while remaining within the kernel ecosystem (retaining access to kernel data structures, routing tables, and existing tools). Three modes:

| Mode | Location | Performance | NIC Requirement |
|------|----------|-------------|-----------------|
| Native XDP | Inside NIC driver | Highest (line rate) | Driver must support XDP |
| Offloaded XDP | Runs ON NIC hardware | Extreme (zero CPU) | Smartnic/programmable NIC |
| Generic XDP | After sk_buff alloc | Lowest (but universal) | Any NIC |

### XDP Hook Point
The specific code location in a NIC driver where an XDP program is invoked, immediately after DMA of a received packet. At this point the packet is in a memory buffer (`xdp_buff`) but no kernel metadata structures have been created yet. This is what makes XDP so fast — it short-circuits the entire normal receive path for packets that should be dropped or redirected.

### XDP Actions
The return codes an XDP program returns to tell the driver what to do with a packet:

| Action | Effect |
|--------|--------|
| `XDP_PASS` | Send packet up to normal kernel network stack |
| `XDP_DROP` | Drop packet immediately (fastest possible drop) |
| `XDP_TX` | Retransmit packet out the same interface (U-turn) |
| `XDP_REDIRECT` | Forward to another interface, CPU, or AF_XDP socket |
| `XDP_ABORTED` | Drop with error trace event (debugging) |

`XDP_REDIRECT` to an AF_XDP socket is the mechanism for high-performance kernel-bypass to userspace while retaining XDP's kernel integration.

---

## Quick Reference Tables

### Performance Tiers

| Approach | Kernel Bypass | Typical Throughput | Latency | Use Case |
|----------|--------------|-------------------|---------|----------|
| Linux stack + iptables | No | ~1 Mpps | 100–500 µs | General purpose |
| iptables + conntrack | No | ~500 Kpps | 200–1000 µs | Legacy NAT/firewall |
| IPVS | No | ~2 Mpps | 50–200 µs | kube-proxy scale-out |
| TC eBPF (Cilium) | Partial | ~3–5 Mpps | 30–100 µs | Container CNI |
| XDP native | Partial | ~10–20 Mpps | 5–30 µs | DDoS, LB, XDP redirect |
| AF_XDP | Partial | ~20–30 Mpps | 2–10 µs | Kernel-bypass to userspace |
| OVS kernel datapath | No | ~3 Mpps | 50–200 µs | OpenStack, SDN |
| OVS-DPDK | Yes | ~20–80 Mpps | 5–20 µs | Telco NFV, high throughput |
| SR-IOV VF passthrough | Yes | Line rate | 1–5 µs | VM/container low latency |
| DPDK + memif | Yes | Line rate | <1 µs | NFV chaining |
| Virtio-net + vhost-user | Yes | ~10–40 Mpps | 3–15 µs | High-perf VM networking |

---

### Kubernetes CNI Plugin Comparison

| Feature | CNI Bridge | Flannel | Calico | Cilium |
|---------|------------|---------|--------|--------|
| Dataplane | Linux Bridge | VXLAN / host-gw | iptables / eBPF / VPP | eBPF |
| Network Policy | No | No | Yes | Yes |
| kube-proxy replacement | No | No | Yes (eBPF mode) | Yes |
| Overlay | No | VXLAN | VXLAN / BGP | VXLAN / BGP |
| Encryption | No | No | WireGuard / IPsec | WireGuard |
| Observability | None | None | Limited | Hubble (rich) |
| Performance | Low | Medium | High | Highest |

---

### Virtual Interface Types

| Interface | Layer | Direction | Kernel Involved | Primary Use |
|-----------|-------|-----------|-----------------|-------------|
| Veth pair | L2 | Bidirectional | Yes | Container ↔ host |
| TAP | L2 | Bidirectional | Yes (FD) | VM Ethernet I/O |
| TUN | L3 | Bidirectional | Yes (FD) | VPN, tunnels |
| Vhost-net | L2 | Bidirectional | Kernel thread | Fast VM networking |
| Vhost-user | L2 | Bidirectional | No (shared mem) | DPDK VM networking |
| Memif | L2 | Bidirectional | No (shared mem) | NFV inter-process |
| Virtio-user PMD | L2 | Bidirectional | No | Container DPDK |
| MACVLAN | L2 | Bidirectional | Yes | Container on physical L2 |
| IPVLAN | L3 | Bidirectional | Yes | High-density pods |
| SR-IOV VF | L2 | Bidirectional | No (HW DMA) | Lowest-latency VM/pod |
| AF_XDP | L2 | Bidirectional | Partial | High-speed userspace |

---

### eBPF Attach Points

| Hook | Location in Stack | Typical Use |
|------|------------------|-------------|
| XDP | NIC driver, pre-sk_buff | DDoS drop, LB, redirect |
| TC ingress | After sk_buff, before routing | Policy, encryption, NAT |
| TC egress | After routing, before NIC | Policy, encryption, stats |
| cgroup/connect4 | Socket connect() syscall | Service LB (Cilium) |
| cgroup/sendmsg | Socket sendmsg() syscall | Transparent proxy |
| socket filter | Socket receive path | Packet filtering |
| kprobe/tracepoint | Kernel function entry/return | Observability |

---

## Architecture Compositions

Understanding how these concepts compose together is more valuable than knowing them in isolation.

**Minimal Kubernetes Pod Networking (CNI Bridge):**
```
Pod NS: eth0 ←─veth─→ host: vethXXX → cni0 (Linux Bridge) → eth0 → physical network
               iptables NAT for Service ClusterIPs (kube-proxy)
```

**Production Kubernetes with Cilium:**
```
Pod NS: eth0 ←─veth─→ TC eBPF hook (policy/encrypt) → routing → NodePort XDP
               Connect4 cgroup hook (socket-level service LB, zero NAT)
               WireGuard encryption for cross-node traffic
               Hubble for flow visibility
```

**Telco NFV Stack:**
```
Physical NIC (SR-IOV PF)
  ├── VF 0 → VFIO → VM A (QEMU + vhost-user) ↔ OVS-DPDK (PMD threads, hugepages, NUMA-aware)
  ├── VF 1 → VFIO → VM B (QEMU + vhost-user) ↔ OVS-DPDK
  └── VF 2 → VFIO → Container (virtio-user PMD) ↔ memif ↔ VPP (5G UPF)
                                OVS-DPDK ↔ DPDK PMD ↔ Physical NIC
```

**eBPF Observability Pipeline:**
```
Network events (drops, flows, errors)
  → eBPF perf ring buffer / BPF maps
  → Cilium agent reads maps
  → Hubble relay aggregates
  → Prometheus metrics + Hubble UI
  → Grafana dashboards
```

---
